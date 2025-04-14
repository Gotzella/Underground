import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(MillionApp());
}

class MillionApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: "Million Dollar App",
      home: LoginPage(),
    );
  }
}

class LoginPage extends StatelessWidget {
  final emailController = TextEditingController();
  final passController = TextEditingController();

  Future<void> login(BuildContext context) async {
    try {
      final cred = await FirebaseAuth.instance.signInWithEmailAndPassword(
        email: emailController.text,
        password: passController.text,
      );
      await giveMillion(cred.user!.uid, cred.user!.email!);
      Navigator.push(context, MaterialPageRoute(builder: (_) => Dashboard()));
    } catch (_) {
      final cred = await FirebaseAuth.instance.createUserWithEmailAndPassword(
        email: emailController.text,
        password: passController.text,
      );
      await giveMillion(cred.user!.uid, cred.user!.email!);
      Navigator.push(context, MaterialPageRoute(builder: (_) => Dashboard()));
    }
  }

  Future<void> giveMillion(String uid, String email) async {
    final userDoc = FirebaseFirestore.instance.collection("users").doc(uid);
    final snapshot = await userDoc.get();
    if (!snapshot.exists) {
      await userDoc.set({
        'email': email,
        'uid': uid,
        'balanceUSD': 1000000.0,
        'receivedMillionGift': true,
        'createdAt': DateTime.now()
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Million Dollar App")),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          children: [
            TextField(controller: emailController, decoration: InputDecoration(labelText: "Email")),
            TextField(controller: passController, decoration: InputDecoration(labelText: "Password"), obscureText: true),
            SizedBox(height: 20),
            ElevatedButton(onPressed: () => login(context), child: Text("Login / Sign Up")),
          ],
        ),
      ),
    );
  }
}

class Dashboard extends StatefulWidget {
  @override
  State<Dashboard> createState() => _DashboardState();
}

class _DashboardState extends State<Dashboard> {
  double balance = 0.0;
  final accountNumController = TextEditingController();
  final bankCodeController = TextEditingController();

  @override
  void initState() {
    super.initState();
    loadBalance();
  }

  Future<void> loadBalance() async {
    final uid = FirebaseAuth.instance.currentUser!.uid;
    final doc = await FirebaseFirestore.instance.collection("users").doc(uid).get();
    setState(() {
      balance = doc.data()?['balanceUSD'] ?? 0.0;
    });
  }

  Future<void> mockTransferUSDToZAR() async {
    final uid = FirebaseAuth.instance.currentUser!.uid;
    final account = accountNumController.text;
    final bank = bankCodeController.text;

    final response = await http.post(
      Uri.parse('https://api.flutterwave.com/v3/transfers'),
      headers: {
        'Authorization': 'Bearer YOUR_FLUTTERWAVE_SECRET_KEY',
        'Content-Type': 'application/json',
      },
      body: jsonEncode({
        "account_bank": bank,
        "account_number": account,
        "amount": balance,
        "currency": "USD",
        "narration": "USD to ZAR transfer",
        "reference": "txn-${DateTime.now().millisecondsSinceEpoch}",
        "callback_url": "https://yourapp.com/transfer/callback"
      }),
    );

    if (response.statusCode == 200) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text("Transfer Successful!")));
    } else {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text("Transfer Failed: ${response.body}")));
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Welcome")),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          children: [
            Text("Balance: \$${balance.toStringAsFixed(2)}", style: TextStyle(fontSize: 28, fontWeight: FontWeight.bold)),
            SizedBox(height: 30),
            TextField(controller: accountNumController, decoration: InputDecoration(labelText: "SA Bank Account Number")),
            TextField(controller: bankCodeController, decoration: InputDecoration(labelText: "Bank Code (e.g. FNB = 250655)")),
            SizedBox(height: 20),
            ElevatedButton(onPressed: mockTransferUSDToZAR, child: Text("Withdraw to SA Bank (USD → ZAR)"))
          ],
        ),
      ),
    );
  }
}
