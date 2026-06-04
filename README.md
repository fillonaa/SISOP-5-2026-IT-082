# Soal 2 - Season OS
Fiorellin Ilona - 082

---

## 1. Penjelasan Soal

Soal 2 meminta praktikan untuk membuat sebuah mini operating system 16-bit sederhana yang berjalan di emulator Bochs. OS ini harus memiliki shell interaktif dengan beberapa command yang bisa dijalankan pengguna.

OS dibangun menggunakan:
- **Assembly x86 16-bit** (`kernel.asm`) untuk low-level bootup dan input karakter
- **C tanpa stdlib** (`kernel.c`) untuk logika shell dan command handler
- **NASM + BCC + LD86** sebagai toolchain compiler
- **Bochs** sebagai emulator untuk menjalankan OS

### Command yang harus diimplementasikan:

| Command | Deskripsi |
|---|---|
| `check` | Menampilkan `ok` |
| `add <a> <b>` | Penjumlahan dua angka |
| `sub <a> <b>` | Pengurangan dua angka |
| `fac <n>` | Faktorial, dengan pesan `know your limit little bro.` jika overflow 16-bit |
| `season <name>` | Mengubah warna teks sesuai musim (`winter`, `spring`, `summer`, `fall`, `radiant`) |
| `triangle <n>` | Mencetak segitiga dari karakter `x` |
| `clear` | Membersihkan layar |
| `help` | Menampilkan daftar command |
| `about` | Menampilkan informasi OS |

### Batasan soal:
- Tidak boleh menggunakan stdlib
- Tidak boleh menggunakan operasi division `/`
- Tidak boleh menggunakan operasi modulo `%`

---

## 2. Step by Step

### A. Install Dependencies

```bash
sudo apt-get update
sudo apt-get install -y nasm bcc bin86 binutils bochs bochs-x
```

### B. Download dan Extract Template

Download file `template.zip` dari link yang diberikan soal, lalu extract:

```bash
cd ~
unzip ~/Downloads/template.zip -d soal_2
cd soal_2/template
ls
```

Setelah extract, struktur folder akan seperti ini:

```
soal_2/
└── template/
    ├── Makefile
    ├── README.md
    ├── bochsrc.txt
    ├── bootloader.asm
    ├── build.sh
    ├── kernel.asm
    └── kernel.c
```

### C. Edit kernel.asm

Buka file dengan nano:

```bash
nano kernel.asm
```

Implementasikan fungsi `_getChar` menggunakan BIOS interrupt `int 0x16`:

```asm
_getChar:
    push bp
    mov bp, sp
    xor ax, ax
    int 0x16
    pop bp
    ret
```

### D. Edit kernel.c

```bash
nano kernel.c
```

Implementasikan semua fungsi helper dan command handler (lihat bagian Kode di bawah).

### E. Fix bochsrc.txt

Cari path BIOS yang tersedia:

```bash
find /usr -name "BIOS-bochs*" 2>/dev/null
find /usr -name "VGABIOS*" 2>/dev/null
```

Edit bochsrc.txt:

```bash
nano bochsrc.txt
```

Isi dengan:

```
megs: 64
romimage: file=/usr/share/bochs/BIOS-bochs-latest
vgaromimage: file=/usr/share/bochs/VGABIOS-lgpl-latest
boot: floppy
floppya: 1_44=floppy.img, status=inserted
floppy_bootsig_check: disabled=1
log: /dev/null
mouse: enabled=0
display_library: x
```

### F. Build

```bash
dd if=/dev/zero of=floppy.img bs=512 count=2880
nasm -f bin bootloader.asm -o bootloader.bin
nasm -f as86 kernel.asm -o kernel-asm.o
bcc -ansi -c kernel.c -o kernel.o
ld86 -0 -d -o kernel.bin kernel-asm.o kernel.o
dd if=bootloader.bin of=floppy.img bs=512 count=1 conv=notrunc
dd if=kernel.bin of=floppy.img bs=512 seek=1 count=15 conv=notrunc
```

### G. Jalankan

```bash
bochs -f bochsrc.txt
```

---

## 3. Kode

### kernel.asm

```asm
bits 16

global _start
global _putInMemory
global _getChar
extern _main

_start:
    cli
    mov ax, cs
    mov ds, ax
    mov es, ax
    sti
    call _main

.hang:
    jmp .hang

_putInMemory:
    push bp
    mov bp, sp
    push ds
    mov ax, [bp+4]
    mov si, [bp+6]
    mov cl, [bp+8]
    mov ds, ax
    mov [si], cl
    pop ds
    pop bp
    ret

_getChar:
    push bp
    mov bp, sp
    xor ax, ax
    int 0x16
    pop bp
    ret
```

### kernel.c

```c
int cursor = 0;
char color = 0x07;

void putInMemory(int segment, int address, char character);
int getChar();

void printChar(char c) {
    putInMemory(0xB800, cursor * 2, c);
    putInMemory(0xB800, cursor * 2 + 1, color);
    cursor++;
}

void newline() {
    int col = cursor;
    while (col >= 80) {
        col = col - 80;
    }
    cursor = cursor + (80 - col);
}

void printString(char *str) {
    int i = 0;
    while (str[i] != '\0') {
        if (str[i] == '\n') {
            newline();
        } else {
            printChar(str[i]);
        }
        i++;
    }
}

void clearScreen() {
    int i;
    for (i = 0; i < 2000; i++) {
        putInMemory(0xB800, i * 2, ' ');
        putInMemory(0xB800, i * 2 + 1, 0x07);
    }
    cursor = 0;
    color = 0x07;
}

void readString(char *buf) {
    int i = 0;
    char c;
    while (1) {
        c = getChar();
        if (c == '\r') {
            buf[i] = '\0';
            break;
        } else if (c == '\b') {
            if (i > 0) {
                i--;
                cursor--;
                putInMemory(0xB800, cursor * 2, ' ');
                putInMemory(0xB800, cursor * 2 + 1, color);
            }
        } else {
            buf[i] = c;
            i++;
            printChar(c);
        }
    }
}

int strcmp(char *a, char *b) {
    int i = 0;
    while (a[i] != '\0' && b[i] != '\0') {
        if (a[i] != b[i]) return 0;
        i++;
    }
    return a[i] == '\0' && b[i] == '\0';
}

int startsWith(char *str, char *prefix) {
    int i = 0;
    while (prefix[i] != '\0') {
        if (str[i] != prefix[i]) return 0;
        i++;
    }
    return 1;
}

int skipSpace(char *cmd, int pos) {
    while (cmd[pos] == ' ') pos++;
    return pos;
}

int getWord(char *cmd, int start, char *out) {
    int i = 0;
    while (cmd[start] != ' ' && cmd[start] != '\0') {
        out[i] = cmd[start];
        i++;
        start++;
    }
    out[i] = '\0';
    return start;
}

int atoi(char *str) {
    int result = 0;
    int i = 0;
    while (str[i] >= '0' && str[i] <= '9') {
        result = result * 10 + (str[i] - '0');
        i++;
    }
    return result;
}

void intToString(int n, char *buf) {
    int i = 0;
    int tmp[10];
    int len = 0;
    int base;
    int quotient;
    int j;
    if (n == 0) { buf[0] = '0'; buf[1] = '\0'; return; }
    if (n < 0) { buf[0] = '-'; i = 1; n = -n; }
    while (n > 0) {
        base = n; quotient = 0;
        while (base >= 10) { base = base - 10; quotient = quotient + 1; }
        tmp[len] = base; len++; n = quotient;
    }
    for (j = 0; j < len; j++) buf[i + j] = '0' + tmp[len - 1 - j];
    buf[i + len] = '\0';
}

int factorial(int n) {
    int result = 1;
    int i;
    for (i = 1; i <= n; i++) {
        result = result * i;
        if (result < 0 || result > 32767) return -1;
    }
    return result;
}

void setSeason(char *name) {
    if (strcmp(name, "winter")) { color = 0x09; printString("winter mode"); }
    else if (strcmp(name, "spring")) { color = 0x0A; printString("spring mode"); }
    else if (strcmp(name, "summer")) { color = 0x0E; printString("summer mode"); }
    else if (strcmp(name, "fall")) { color = 0x06; printString("fall mode"); }
    else if (strcmp(name, "radiant")) { color = 0x0D; printString("radiant mode"); }
    else { printString("unknown season"); }
}

void printTriangle(int n) {
    int i, j;
    for (i = 1; i <= n; i++) {
        for (j = 0; j < i; j++) printChar('x');
        newline();
    }
}

void main() {
    char cmd[64];
    char arg1[16];
    char arg2[16];
    int pos;
    int a, b, result;
    char resBuf[12];

    clearScreen();
    printString("Welcome to Assistant's Last Gift");
    newline();
    printString("type 'help'");
    newline();
    newline();

    while (1) {
        color = 0x07;
        printString("> ");
        readString(cmd);
        newline();

        if (strcmp(cmd, "check")) {
            printString("ok");
        } else if (startsWith(cmd, "add ")) {
            pos = skipSpace(cmd, 4); pos = getWord(cmd, pos, arg1);
            pos = skipSpace(cmd, pos); getWord(cmd, pos, arg2);
            a = atoi(arg1); b = atoi(arg2); result = a + b;
            intToString(result, resBuf); printString(resBuf);
        } else if (startsWith(cmd, "sub ")) {
            pos = skipSpace(cmd, 4); pos = getWord(cmd, pos, arg1);
            pos = skipSpace(cmd, pos); getWord(cmd, pos, arg2);
            a = atoi(arg1); b = atoi(arg2); result = a - b;
            intToString(result, resBuf); printString(resBuf);
        } else if (startsWith(cmd, "fac ")) {
            pos = skipSpace(cmd, 4); getWord(cmd, pos, arg1);
            a = atoi(arg1); result = factorial(a);
            if (result == -1) { printString("know your limit little bro."); }
            else { intToString(result, resBuf); printString(resBuf); }
        } else if (startsWith(cmd, "season ")) {
            pos = skipSpace(cmd, 7); getWord(cmd, pos, arg1);
            setSeason(arg1);
        } else if (startsWith(cmd, "triangle ")) {
            pos = skipSpace(cmd, 9); getWord(cmd, pos, arg1);
            a = atoi(arg1); printTriangle(a);
        } else if (strcmp(cmd, "clear")) {
            clearScreen();
        } else if (strcmp(cmd, "help")) {
            printString("check add sub fac season triangle clear about");
        } else if (strcmp(cmd, "about")) {
            printString("Season OS v1.0 by IT-082");
        } else if (cmd[0] != '\0') {
            printString("unknown command");
        }
        newline();
    }
}
```

---

## 4. Kendala

Bochs layar hitam (black screen)**

Setelah build berhasil, Bochs menampilkan layar hitam dan kernel tidak menampilkan output apapun. Sudah dicoba mengetik `c` di prompt `<bochs>` untuk continue, namun layar tetap hitam. Diduga kernel tidak berhasil dieksekusi karena ada bug di `kernel.asm` akibat typo pada instruksi `xor`.

<img width="879" height="673" alt="WhatsApp Image 2026-06-04 at 19 42 47 (1)" src="https://github.com/user-attachments/assets/b528bbfb-d0f4-4130-9217-d9ec6c7b91dc" />
