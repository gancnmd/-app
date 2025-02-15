// main.dart - 客户端主界面
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'chat_screen.dart';

void main() => runApp(SocialApp());

class SocialApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: '简易社交APP',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: StreamBuilder<User?>(
        stream: FirebaseAuth.instance.authStateChanges(),
        builder: (context, snapshot) {
          return snapshot.hasData ? HomeScreen() : LoginScreen();
        },
      ),
    );
  }
}

// 登录注册界面
class LoginScreen extends StatefulWidget {
  @override
  _LoginScreenState createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();

  Future<void> _signIn() async {
    try {
      await FirebaseAuth.instance.signInWithEmailAndPassword(
        email: _emailController.text,
        password: _passwordController.text,
      );
    } catch (e) {
      print("登录失败: $e");
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('登录/注册')),
      body: Padding(
        padding: EdgeInsets.all(20),
        child: Column(
          children: [
            TextField(controller: _emailController, decoration: InputDecoration(labelText: '邮箱')),
            TextField(controller: _passwordController, obscureText: true, decoration: InputDecoration(labelText: '密码')),
            ElevatedButton(onPressed: _signIn, child: Text('进入'))
          ],
        ),
      ),
    );
  }
}

// 主功能界面
class HomeScreen extends StatelessWidget {
  final _firestore = FirebaseFirestore.instance;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('社交主页')),
      body: Column(
        children: [
          // 好友列表
          Expanded(
            child: StreamBuilder<QuerySnapshot>(
              stream: _firestore.collection('users').snapshots(),
              builder: (context, snapshot) {
                if (!snapshot.hasData) return CircularProgressIndicator();
                return ListView.builder(
                  itemCount: snapshot.data!.docs.length,
                  itemBuilder: (context, index) {
                    var user = snapshot.data!.docs[index];
                    return ListTile(
                      title: Text(user['email']),
                      onTap: () => Navigator.push(
                        context,
                        MaterialPageRoute(builder: (context) => ChatScreen(targetUser: user)),
                      ),
                    );
                  },
                );
              },
            ),
          ),
          // 发布动态
          Padding(
            padding: EdgeInsets.all(8),
            child: TextField(
              onSubmitted: (text) {
                _firestore.collection('posts').add({
                  'content': text,
                  'author': FirebaseAuth.instance.currentUser!.uid,
                  'timestamp': FieldValue.serverTimestamp(),
                });
              },
              decoration: InputDecoration(hintText: '分享新鲜事...'),
            ),
          )
        ],
      ),
    );
  }
}


// chat_screen.dart - 聊天界面
import 'package:flutter/material.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

class ChatScreen extends StatelessWidget {
  final DocumentSnapshot targetUser;

  ChatScreen({required this.targetUser});

  final _messageController = TextEditingController();
  final _firestore = FirebaseFirestore.instance;

  void _sendMessage() {
    if (_messageController.text.isEmpty) return;
    
    _firestore.collection('messages').add({
      'text': _messageController.text,
      'sender': FirebaseAuth.instance.currentUser!.uid,
      'receiver': targetUser.id,
      'timestamp': FieldValue.serverTimestamp(),
    });
    _messageController.clear();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(targetUser['email'])),
      body: Column(
        children: [
          Expanded(
            child: StreamBuilder<QuerySnapshot>(
              stream: _firestore
                  .collection('messages')
                  .where('sender', whereIn: [
                    FirebaseAuth.instance.currentUser!.uid,
                    targetUser.id
                  ])
                  .orderBy('timestamp')
                  .snapshots(),
              builder: (context, snapshot) {
                if (!snapshot.hasData) return Center(child: CircularProgressIndicator());
                
                return ListView.builder(
                  itemCount: snapshot.data!.docs.length,
                  itemBuilder: (context, index) {
                    var message = snapshot.data!.docs[index];
                    bool isMe = message['sender'] == FirebaseAuth.instance.currentUser!.uid;
                    
                    return Align(
                      alignment: isMe ? Alignment.centerRight : Alignment.centerLeft,
                      child: Container(
                        margin: EdgeInsets.all(8),
                        padding: EdgeInsets.all(12),
                        decoration: BoxDecoration(
                          color: isMe ? Colors.blue : Colors.grey,
                          borderRadius: BorderRadius.circular(12),
                        ),
                        child: Text(message['text'], 
                          style: TextStyle(color: Colors.white)),
                      ),
                    );
                  },
                );
              },
            ),
          ),
          Padding(
            padding: EdgeInsets.all(8),
            child: Row(
              children: [
                Expanded(
                  child: TextField(
                    controller: _messageController,
                    decoration: InputDecoration(hintText: '输入消息...'),
                  ),
                ),
                IconButton(
                  icon: Icon(Icons.send),
                  onPressed: _sendMessage,
                ),
              ],
            ),
          )
        ],
      ),
    );
  }
}

flutter create social_app
cd social_app
flutter pub add firebase_core cloud_firestore firebase_auth# -app
