1. Struktur Proyek
Proyek terdiri dari 5 file utama:

text
lib/
├── main.dart                  # Entry point, tema aplikasi
├── models/
│   └── task.dart              # Model data Task
├── pages/
│   ├── home_page.dart         # Halaman utama (daftar tugas + API quote)
│   ├── add_task_page.dart     # Form tambah tugas
│   └── edit_task_page.dart    # Form edit tugas
└── widgets/
    └── task_item.dart         # Widget tampilan satu tugas (Card + gesture)
Pemisahan file ini membuat kode lebih rapi dan mudah dipelihara.

2. Model Data (task.dart)
dart
class Task {
  final UniqueKey id;
  String nama;
  DateTime deadline;
  String prioritas;
  String deskripsi;
  bool isDone;
  ...
}
Setiap tugas memiliki properti: id unik, nama, deadline (tanggal), prioritas (Rendah/Sedang/Tinggi), deskripsi, dan status isDone. Model ini digunakan di seluruh halaman untuk menyimpan dan mengirim data antar halaman.

3. Halaman Utama (home_page.dart) – Pusat Navigasi & API
a. State & List Tugas
dart
List<Task> tasks = [];
late Future<Map<String, dynamic>> quoteFuture;
tasks menyimpan daftar tugas di memori (stateful widget).

quoteFuture adalah Future yang akan diisi hasil API.

b. API Consumption + FutureBuilder
Fungsi fetchQuote:

dart
Future<Map<String, dynamic>> fetchQuote() async {
  final response = await http.get(Uri.parse('https://dummyjson.com/quotes/random'));
  if (response.statusCode == 200) {
    final jsonResponse = json.decode(response.body);
    return {'content': jsonResponse['quote'], 'author': jsonResponse['author']};
  } else {
    throw Exception('Gagal mengambil quote');
  }
}
Menggunakan package http untuk GET request.

Mengolah JSON menjadi Map dengan key 'content' dan 'author'.

FutureBuilder di build:

dart
FutureBuilder<Map<String, dynamic>>(
  future: quoteFuture,
  builder: (context, snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return CircularProgressIndicator();
    } else if (snapshot.hasError) {
      return Column(...); // tampilkan error + tombol refresh
    } else if (snapshot.hasData) {
      return Card(
        child: Text('"${snapshot.data!['content']}" - ${snapshot.data!['author']}'),
      );
    }
    return SizedBox();
  },
)
connectionState.waiting → tampilkan loading.

hasError → tampilkan pesan dan tombol refresh (memanggil refreshQuote()).

hasData → tampilkan quote dalam Card.

Ini memenuhi poin API consumption dan FutureBuilder.

c. ListView untuk Menampilkan Tugas
dart
Expanded(
  child: tasks.isEmpty
    ? Center(child: Text('Belum ada tugas...'))
    : ListView.builder(
        itemCount: tasks.length,
        itemBuilder: (context, index) {
          return TaskItem(...);
        },
      ),
)
Jika tasks kosong, tampilkan teks pesan.

Jika ada, gunakan ListView.builder untuk membuat widget TaskItem setiap baris.

d. FloatingActionButton (Navigasi ke Tambah)
dart
floatingActionButton: FloatingActionButton(
  onPressed: () async {
    final newTask = await Navigator.push<Task>(
      context,
      MaterialPageRoute(builder: (_) => AddTaskPage()),
    );
    if (newTask != null) _addTask(newTask);
  },
  child: Icon(Icons.add),
)
Navigasi ke AddTaskPage menggunakan Navigator.push.

Tunggu hasil kembalian (await), jika ada task baru, tambahkan ke tasks.

e. Fungsi CRUD
_addTask(): menambah task ke list, lalu mengurutkan berdasarkan deadline.

_editTask(): mengganti task dengan id yang sama, lalu mengurutkan ulang.

_deleteTask(): menghapus task berdasarkan id.

_toggleTaskStatus(): mengubah status checkbox.

Semua operasi memanggil setState() agar UI terupdate.

4. Halaman Tambah Tugas (add_task_page.dart) – Form & Validasi
a. Form Key dan Controller
dart
final _formKey = GlobalKey<FormState>();
final _namaController = TextEditingController();
final _deskripsiController = TextEditingController();
DateTime? _selectedDate;
String _selectedPriority = 'Sedang';
b. Field dengan Validasi
dart
TextFormField(
  controller: _namaController,
  validator: (value) {
    if (value == null || value.trim().isEmpty) return 'Nama tugas tidak boleh kosong';
    return null;
  },
)
Validasi wajib isi.

c. DatePicker dengan Validasi (tidak boleh kurang dari hari ini)
dart
Future<void> _selectDate(BuildContext context) async {
  final picked = await showDatePicker(
    context: context,
    firstDate: DateTime.now(), // 🔴 mencegah tanggal lampau
    lastDate: DateTime.now().add(Duration(days: 365*5)),
  );
  if (picked != null) setState(() => _selectedDate = picked);
}
firstDate: DateTime.now() memastikan user tidak bisa memilih tanggal kemarin.

d. Dropdown untuk Prioritas
dart
DropdownButtonFormField<String>(
  value: _selectedPriority,
  items: ['Rendah', 'Sedang', 'Tinggi'].map(...).toList(),
  onChanged: (newValue) => setState(() => _selectedPriority = newValue!),
)
e. Tombol Batal & Simpan (ElevatedButton + OutlinedButton)
dart
Row(children: [
  Expanded(child: OutlinedButton(onPressed: () => Navigator.pop(context), child: Text('Batal'))),
  SizedBox(width: 16),
  Expanded(child: ElevatedButton(onPressed: _saveTask, child: Text('Simpan'))),
])
OutlinedButton: batalkan dan kembali tanpa mengirim data.

ElevatedButton: validasi form, lalu Navigator.pop(context, newTask) untuk mengirim task baru ke halaman utama.

Poin: Memenuhi persyaratan minimal 2 jenis button dan validasi form.

5. Halaman Edit Tugas (edit_task_page.dart)
Mirip dengan tambah, namun:

Menerima parameter Task task dari konstruktor.

Di initState, mengisi controller dengan data task yang ada:

dart
_namaController.text = widget.task.nama;
_selectedDate = widget.task.deadline;
Setelah edit, mengirim task yang sudah diperbarui kembali ke halaman utama dengan Navigator.pop(context, updatedTask).

6. Widget TaskItem (task_item.dart) – GestureDetector, Checkbox, IconButton
a. GestureDetector untuk Interaksi
dart
GestureDetector(
  onTap: () => _showTaskDetails(context),   // tekan sekali → detail dialog
  onLongPress: () => _confirmDelete(context), // tekan lama → konfirmasi hapus
  child: Card(...),
)
Memenuhi poin GestureDetector dengan dua aksi berbeda.

b. Checkbox untuk Tandai Selesai
dart
Checkbox(
  value: task.isDone,
  onChanged: onChanged, // callback dari home_page
)
c. IconButton Edit & Hapus
dart
trailing: Row(
  children: [
    IconButton(icon: Icon(Icons.edit), onPressed: onEdit),
    IconButton(icon: Icon(Icons.delete), onPressed: () => _confirmDelete(context)),
  ],
)
Edit memicu navigasi ke EditTaskPage.

Delete memunculkan dialog konfirmasi.

d. Fitur Tambahan (Nilai Plus)
Warna prioritas : hijau (rendah), orange (sedang), merah (tinggi).

Strikethrough pada nama tugas jika isDone == true.

Deadline terlewat: teks deadline berwarna merah jika deadline < today dan belum selesai.

Countdown sisa hari: "3 hari lagi" atau "Terlewat 2 hari".

Contoh kode countdown:

dart
final difference = taskDate.difference(today).inDays;
if (difference == 0) countdownText = "Hari ini";
else if (difference > 0) countdownText = "$difference hari lagi";
else countdownText = "Terlewat ${difference.abs()} hari";
7. Alur Navigasi Lengkap
Halaman Utama → Tambah Tugas
Klik FAB → Navigator.push ke AddTaskPage → isi form → klik Simpan → Navigator.pop(context, newTask) → halaman utama menerima dan menambah ke tasks.

Halaman Utama → Edit Tugas
Klik tombol edit di TaskItem → Navigator.push ke EditTaskPage dengan membawa data task → edit → Simpan → pop dengan task yang sudah diubah → halaman utama update.

Hapus Tugas
Tekan lama (atau klik icon delete) → dialog konfirmasi → panggil onDelete() → hapus dari list.

Kembali tanpa perubahan
Tombol Batal pada form akan Navigator.pop(context) tanpa mengirim data.

8. Ringkasan Pemenuhan Poin Praktikum
Poin	Implementasi
Navigasi	Navigator.push dan pop antar halaman (Home, Add, Edit).
Form & Validasi	TextFormField dengan validator, showDatePicker dengan firstDate mencegah tanggal lampau.
Minimal 2 jenis button	ElevatedButton (Simpan), OutlinedButton (Batal), IconButton (Edit/Hapus), FloatingActionButton (Tambah).
GestureDetector	onTap → dialog detail, onLongPress → konfirmasi hapus.
API + FutureBuilder	Mengambil quote dari dummyjson.com dengan http.get, ditampilkan dengan FutureBuilder (loading, error, sukses).
ListView	ListView.builder di halaman utama menampilkan TaskItem dari list tasks.
9. Cara Menjalankan
Buat project Flutter baru: flutter create my_task

Tambahkan dependency di pubspec.yaml:

yaml
dependencies:
  http: ^1.1.0
  intl: ^0.18.1
Ganti semua file sesuai kode yang diberikan.

Jalankan flutter pub get

flutter run

Pastikan perangkat atau emulator terhubung internet.
