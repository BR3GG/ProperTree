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
// 预设设备型号
   _buildDropdown('设备型号', ['FT-991A', 'IC-7300', 'FT-DX10'], (v) => selectedRadio = v)
   
   // 预设天线型号
   _buildDropdown('天线型号', ['EFHW-4010', 'YP-3', 'Hexbeam'], (v) => selectedAntenna = v)
// RST输入验证正则
   final regex = RegExp(r'^[1-5][1-9][1-9](/[1-9][1-9])?$');
   
   // 输入格式示例
   - 有效格式：599、599/59、339/33
   - 无效格式：60A、5999、5X9
CREATE TABLE qso_log (
     id INTEGER PRIMARY KEY AUTOINCREMENT,
     time TEXT NOT NULL,         -- ISO8601时间
     callsign TEXT NOT NULL,     -- 对方呼号
     radio TEXT NOT NULL,        -- 设备型号
     antenna TEXT NOT NULL,      -- 天线型号
     rst_tx TEXT NOT NULL,       -- 发送报告
     rst_rx TEXT NOT NULL        -- 接收报告
   )
// 在_showAddDialog方法中更新选项列表
   _buildDropdown('设备型号', [
     'FT-891', 
     'IC-705',
     'TS-590SG',
     'KX3'
   ], (v) => selectedRadio = v)
// 添加导出按钮
   IconButton(
     icon: Icon(Icons.export),
     onPressed: () => _exportADIF(),
   )

   // ADIF导出实现
   void _exportADIF() async {
     final qsos = await _dbHelper.getQSOs();
     final adif = qsos.map((qso) => '''
       <CALL:${qso['callsign'].length}>${qso['callsign']}
       <QSO_DATE:8>${DateTime.parse(qso['time']).toString().substring(0,10).replaceAll('-', '')}
       <TIME_ON:4>${DateTime.parse(qso['time']).toString().substring(11,13)}${DateTime.parse(qso['time']).toString().substring(14,16)}
       <RIG:${qso['radio'].length}>${qso['radio']}
       <ANTENNA:${qso['antenna'].length}>${qso['antenna']}
       <RST_SENT:${qso['rst_tx'].length}>${qso['rst_tx']}
       <RST_RCVD:${qso['rst_rx'].length}>${qso['rst_rx']}
       <EOR>
     ''').join('\n');
     
     // 保存文件逻辑...
   }
// 修改后的数据库结构
class DatabaseHelper {
  // ...
  static final _table = 'qso_log';

  Future<Database> _initDB() async {
    // ...
    onCreate: (db, version) => db.execute('''
      CREATE TABLE $_table (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        time TEXT NOT NULL,
        callsign TEXT NOT NULL,
        freq REAL NOT NULL,
        radio TEXT NOT NULL,
        antenna TEXT NOT NULL,
        rst_main TEXT NOT NULL,
        rst_sub TEXT,
        rst_tx TEXT NOT NULL,
        rst_rx TEXT NOT NULL
      )
    ''')
    // ...
  }
}

// 修改后的输入表单
Widget _buildQSOForm() {
  final freqCtrl = TextEditingController();
  final rstMainCtrl = TextEditingController(text: '599');
  final rstSubCtrl = TextEditingController(text: '59');

  return Column(
    children: [
      TextFormField(
        controller: freqCtrl,
        decoration: InputDecoration(labelText: '频率 (MHz)'),
        keyboardType: TextInputType.numberWithOptions(decimal: true),
        validator: (v) => _validateFrequency(v),
      ),
      SizedBox(height: 16),
      Row(
        children: [
          Expanded(
            child: TextFormField(
              controller: rstMainCtrl,
              decoration: InputDecoration(labelText: '主报告 (R/S/T)'),
              validator: (v) => _validateRSTMain(v)),
          ),
          SizedBox(width: 16),
          Expanded(
            child: TextFormField(
                controller: rstSubCtrl,
                decoration: InputDecoration(labelText: '分报告 (额外信息)'),
                validator: (v) => _validateRSTSub(v)),
          ),
        ],
      ),
      // 保留原有的TX/RX报告栏
    ],
  );
}

// 验证方法
String? _validateFrequency(String? value) {
  final freq = double.tryParse(value ?? '');
  return (freq != null && freq >= 0.1 && freq <= 148) ? null : '无效频率 (0.1-148MHz)';
}

String? _validateRSTMain(String? value) {
  return RegExp(r'^[1-5][1-9][1-9]$').hasMatch(value ?? '') 
      ? null : '格式错误（例：599）';
}

String? _validateRSTSub(String? value) {
  return (value?.isEmpty ?? true) || RegExp(r'^[0-9A-Za-z]{1,4}$').hasMatch(value!)
      ? null : '最多4位字母数字';
}
android/app/src/main/res/
     mipmap-hdpi/ic_launcher.png (72x72)
     mipmap-mdpi/ic_launcher.png (48x48)
     mipmap-xhdpi/ic_launcher.png (96x96)
     mipmap-xxhdpi/ic_launcher.png (144x144)
     mipmap-xxxhdpi/ic_launcher.png (192x192)
# pubspec.yaml
   dev_dependencies:
     flutter_native_splash: ^2.3.1

   flutter_native_splash:
     color: "#2c3e50"
     image: "assets/splash.png"
     android: true
     ios: true
// 备份恢复服务类
class BackupService {
  static Future<void> backupData(BuildContext context) async {
    final dbPath = await DatabaseHelper().dbPath;
    final dir = await getExternalStorageDirectory();
    final backupFile = File('${dir?.path}/qsolog_backup_${DateTime.now().millisecondsSinceEpoch}.db');
    
    await backupFile.writeAsBytes(await File(dbPath).readAsBytes());
    
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('备份成功: ${backupFile.path}'))
    );
  }

  static Future<void> restoreData(BuildContext context) async {
    final filePickerResult = await FilePicker.platform.pickFiles(
      type: FileType.custom,
      allowedExtensions: ['db'],
    );

    if (filePickerResult != null) {
      final selectedFile = File(filePickerResult.files.single.path!);
      final dbPath = await DatabaseHelper().dbPath;
      
      await selectedFile.copy(dbPath);
      
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('恢复成功，请重启应用'))
      );
    }
  }
}

// 在设置界面添加操作按钮
ListTile(
  leading: Icon(Icons.backup),
  title: Text('备份数据'),
  onTap: () => BackupService.backupData(context),
),
ListTile(
  leading: Icon(Icons.restore),
  title: Text('恢复数据'),
  onTap: () => BackupService.restoreData(context),
),
// 更新检查服务
class AppUpdater {
  static final _currentVersion = '1.0.0';
  static final _updateUrl = 'https://your-api.com/latest-version';

  static Future<void> checkUpdate(BuildContext context) async {
    try {
      final response = await http.get(Uri.parse(_updateUrl));
      final latest = jsonDecode(response.body);
      
      if (latest['version'] != _currentVersion) {
        showDialog(
          context: context,
          builder: (_) => AlertDialog(
            title: Text('发现新版本 ${latest['version']}'),
            content: Text(latest['releaseNotes']),
            actions: [
              TextButton(
                child: Text('稍后再说'),
                onPressed: () => Navigator.pop(context)),
              TextButton(
                child: Text('立即更新'),
                onPressed: () => _launchUpdate(latest['downloadUrl'])),
            ],
          ),
        );
      }
    } catch (e) {
      print('更新检查失败: $e');
    }
  }

  static void _launchUpdate(String url) async {
    if (await canLaunch(url)) {
      await launch(url);
    }
  }
}

// 在应用启动时检查
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(QSOApp());
  await AppUpdater.checkUpdate();
}
// 主界面增强
class _QSOMainScreenState extends State<QSOMainScreen> {
  // ...
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Ham Logger'),
        actions: [
          IconButton(
            icon: Icon(Icons.settings),
            onPressed: () => _showSettings(context),
          ),
        ],
      ),
      // ...
    );
  }

  void _showSettings(BuildContext context) {
    showModalBottomSheet(
      context: context,
      builder: (_) => Column(
        children: [
          ListTile(
            title: Text('数据管理'),
            leading: Icon(Icons.storage),
          ),
          Divider(),
          BackupRestoreButtons(),
          ListTile(
            title: Text('检查更新'),
            leading: Icon(Icons.update),
            onTap: () => AppUpdater.checkUpdate(context),
          ),
        ],
      ),
    );
  }
}

// 记录卡片显示增强
class QSOCard extends StatelessWidget {
  // ...
  
  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: EdgeInsets.all(12),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(qso['callsign'], style: TextStyle(fontSize: 18)),
            SizedBox(height: 8),
            _buildInfoRow(Icons.access_time, 
              DateFormat('MM-dd HH:mm').format(time)),
            _buildInfoRow(Icons.wifi, 
              '${qso['freq']}MHz | ${qso['radio']}'),
            _buildInfoRow(Icons.satellite, 
              'TX: ${qso['rst_tx']} RX: ${qso['rst_rx']}'),
            if (qso['rst_sub']?.isNotEmpty ?? false)
              _buildInfoRow(Icons.note, '备注: ${qso['rst_sub']}'),
          ],
        ),
      ),
    );
  }

  Widget _buildInfoRow(IconData icon, String text) {
    return Padding(
      padding: EdgeInsets.symmetric(vertical: 4),
      child: Row(
        children: [
          Icon(icon, size: 16),
          SizedBox(width: 8),
          Text(text),
        ],
      ),
    );
  }
}
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
   <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
   <uses-permission android:name="android.permission.INTERNET" />
<key>NSAppTransportSecurity</key>
   <dict>
     <key>NSAllowsArbitraryLoads</key>
     <true/>
   </dict>
{
     "version": "1.1.0",
     "releaseNotes": "新增频率输入和备份功能",
     "downloadUrl": "https://example.com/app-release.apk"
   }
// 有效组合
   (rst_main, rst_sub) => ('599', '59')
   ('339', '33A')
   
   // 无效组合
   ('59', '')  // 主报告必须3位
   ('5X9', '') // 包含字母
# 在.git/hooks/post-commit中添加
#!/bin/sh
flutter build apk --release
cp build/app/outputs/apk/release/app-release.apk ~/HamRadioLog/
echo "新版本已生成至家目录"
# 每日凌晨自动备份到Google Drive
0 3 * * * /usr/bin/rclone copy ~/HamRadioLog/backups/ mydrive:/HamLogBackup
# 配置同步目录（支持全平台）
syncthing
# 访问 http://localhost:8384 配置同步文件夹
# 生成签名密钥
openssl genpkey -algorithm RSA -out private_key.pem
openssl rsa -pubout -in private_key.pem -out public_key.pem

# 应用内验证示例（伪代码）
bool verifyApk(File apk, String signature) {
  final publicKey = loadPublicKey('public_key.pem');
  return verifyData(apk.readAsBytes(), base64Decode(signature), publicKey);
}
sequenceDiagram
用户手机->>+本地服务器: 检查更新
本地服务器-->>-用户手机: 返回最新版本号
用户手机->>+Minio存储: 下载APK
Minio存储-->>-用户手机: 返回安装包
用户手机->>验证模块: 校验签名
验证模块-->>安装流程: 验证
graph TD
A[每日3AM] --> B{是否插入电源}
B -->|是| C[执行本地备份]
C --> D[同步到Google Drive]
D --> E[发送备份成功通知]
# 安装必要组件
pkg install termux-api openssh

# 自动安装脚本
#!/data/data/com.termux/files/usr/bin/bash
wget http://local-pc:8080/latest.apk
termux-open latest.apk
# 每日检查任务
termux-job-scheduler \
  --job-id 1 \
  --period-ms 86400000 \
  --script ~/check_update.sh

