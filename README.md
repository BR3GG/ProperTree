import 'package:flutter/material.dart';
import 'package:path/path.dart';
import 'package:sqflite/sqflite.dart';

void main() => runApp(QSOApp());

class QSOApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Ham Radio Logger',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: QSOMainScreen(),
    );
  }
}

class QSOMainScreen extends StatefulWidget {
  @override
  _QSOMainScreenState createState() => _QSOMainScreenState();
}

class _QSOMainScreenState extends State<QSOMainScreen> {
  final _dbHelper = DatabaseHelper();
  List<Map<String, dynamic>> _qsos = [];

  @override
  void initState() {
    super.initState();
    _loadQSOs();
  }

  void _loadQSOs() async {
    final data = await _dbHelper.getQSOs();
    setState(() => _qsos = data);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('通联日志'), 
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () => _showAddDialog(context),
      ),
      body: ListView.builder(
        itemCount: _qsos.length,
        itemBuilder: (ctx, i) => QSOCard(qso: _qsos[i]),
      ),
    );
  }

  void _showAddDialog(BuildContext context) {
    final formKey = GlobalKey<FormState>();
    final callsignCtrl = TextEditingController();
    String? selectedRadio = 'FT-991A';
    String? selectedAntenna = 'EFHW-4010';
    final rstTxCtrl = TextEditingController(text: '599');
    final rstRxCtrl = TextEditingController(text: '599');

    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text('新建通联记录'),
        content: Form(
          key: formKey,
          child: SingleChildScrollView(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                TextFormField(
                  controller: callsignCtrl,
                  decoration: InputDecoration(labelText: '对方呼号'),
                  validator: (v) => v!.isEmpty ? '必须填写呼号' : null,
                  onChanged: (_) => formKey.currentState?.validate(),
                ),
                SizedBox(height: 16),
                _buildDropdown('设备型号', ['FT-991A', 'IC-7300', 'FT-DX10'], 
                    (v) => selectedRadio = v),
                _buildDropdown('天线型号', ['EFHW-4010', 'YP-3', 'Hexbeam'], 
                    (v) => selectedAntenna = v),
                SizedBox(height: 16),
                Row(
                  children: [
                    Expanded(
                      child: TextFormField(
                        controller: rstTxCtrl,
                        decoration: InputDecoration(labelText: '发送RST'),
                        validator: _validateRST,
                      ),
                    ),
                    SizedBox(width: 16),
                    Expanded(
                      child: TextFormField(
                        controller: rstRxCtrl,
                        decoration: InputDecoration(labelText: '接收RST'),
                        validator: _validateRST,
                      ),
                    ),
                  ],
                ),
              ],
            ),
          ),
        ),
        actions: [
          TextButton(
            child: Text('保存'),
            onPressed: () async {
              if (formKey.currentState!.validate()) {
                await _dbHelper.insertQSO({
                  'time': DateTime.now().toIso8601String(),
                  'callsign': callsignCtrl.text.toUpperCase(),
                  'radio': selectedRadio,
                  'antenna': selectedAntenna,
                  'rst_tx': rstTxCtrl.text,
                  'rst_rx': rstRxCtrl.text,
                });
                _loadQSOs();
                Navigator.pop(context);
              }
            },
          )
        ],
      ),
    );
  }

  Widget _buildDropdown(String label, List<String> options, 
      ValueChanged<String?> onChanged) {
    return DropdownButtonFormField<String>(
      decoration: InputDecoration(labelText: label),
      value: options.first,
      items: options.map((e) => 
          DropdownMenuItem(value: e, child: Text(e))).toList(),
      onChanged: onChanged,
    );
  }

  String? _validateRST(String? value) {
    final regex = RegExp(r'^[1-5][1-9][1-9](/[1-9][1-9])?$');
    return regex.hasMatch(value ?? '') ? null : '格式错误（例：599/59）';
  }
}

class QSOCard extends StatelessWidget {
  final Map<String, dynamic> qso;

  QSOCard({required this.qso});

  @override
  Widget build(BuildContext context) {
    final time = DateTime.parse(qso['time']);
    return Card(
      child: ListTile(
        title: Text(qso['callsign']),
        subtitle: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('${time.year}-${time.month}-${time.day} ${time.hour}:${time.minute}'),
            Text('设备：${qso['radio']}  天线：${qso['antenna']}'),
            Text('RST: TX ${qso['rst_tx']} / RX ${qso['rst_rx']}'),
          ],
        ),
      ),
    );
  }
}

class DatabaseHelper {
  static final _dbName = 'qsolog.db';
  static final _table = 'qso_log';

  Future<Database> _initDB() async {
    final path = join(await getDatabasesPath(), _dbName);
    return openDatabase(
      path,
      version: 1,
      onCreate: (db, version) => db.execute('''
        CREATE TABLE $_table (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          time TEXT NOT NULL,
          callsign TEXT NOT NULL,
          radio TEXT NOT NULL,
          antenna TEXT NOT NULL,
          rst_tx TEXT NOT NULL,
          rst_rx TEXT NOT NULL
        )
      '''),
    );
  }

  Future<int> insertQSO(Map<String, dynamic> qso) async {
    final db = await _initDB();
    return db.insert(_table, qso);
  }

  Future<List<Map<String, dynamic>>> getQSOs() async {
    final db = await _initDB();
    return db.query(_table, orderBy: 'time DESC');
  }
}
