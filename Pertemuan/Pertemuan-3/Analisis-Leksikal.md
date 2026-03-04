# Analisis Leksikal

## Tujuan Pembelajaran

- Memahami peran analisis leksikal dalam proses kompilasi.
- Menjelaskan konsep token, lexeme, dan pattern.
- Mempelajari hubungan antara ekspresi reguler, NFA, dan DFA.
- Mengimplementasikan lexer sederhana (hand-written dan tool-based).
- Menangani konflik token, buffering input, dan error leksikal.

---

## 1. Pengantar

Analisis leksikal (lexical analysis) adalah tahap awal dalam proses kompilasi yang mengubah urutan karakter input (program sumber) menjadi urutan token yang bermakna untuk sintaksis selanjutnya. Komponen ini disebut juga lexer atau scanner.

Peran utamanya:
- Membuang whitespace dan komentar yang tidak relevan untuk parsing.
- Mengelompokkan karakter menjadi token (mis. identifier, angka, operator).
- Memberikan atribut pada token (nilai numerik, string literal, posisi baris/kolom).

Hasil keluaran: stream token seperti (TOKEN_TYPE, lexeme, atribut).

## 2. Konsep Dasar: Token, Lexeme, Pattern

- Token: kategori simbol terminal (contoh: IDENTIFIER, NUMBER, PLUS, IF, INT_LITERAL).
- Lexeme: urutan karakter aktual yang cocok dengan token tertentu (mis. `count`, `123`).
- Pattern: deskripsi formal dari lexeme, biasanya dengan ekspresi reguler.

Contoh: untuk bahasa sederhana,

- Pattern IDENTIFIER: `[A-Za-z_][A-Za-z0-9_]*` → lexeme `foo`, `bar2`
- Pattern INTEGER: `[0-9]+` → lexeme `0`, `42`
- Pattern WHITESPACE: `[ \t\r\n]+` (biasanya dibuang)

## 3. Ekspresi Reguler dan Bahasa Reguler

Ekspresi reguler (regular expressions) adalah alat yang umum untuk mendefinisikan pola token. Secara formal, ekspresi reguler R dibangun dari operasi dasar:

$$
R ::= \emptyset \mid \epsilon \mid a \mid (R_1 R_2) \mid (R_1 | R_2) \mid (R)^*
$$

Contoh notasi praktis (POSIX-like): `a`, `ab*`, `(if|else)`.

Ekspresi reguler merepresentasikan bahasa reguler yang dapat diimplementasikan menggunakan automata hingga (finite automata).

## 4. NFA dan DFA

- NFA (Nondeterministic Finite Automaton): mudah dibangun dari ekspresi reguler (Thompson's construction).
- DFA (Deterministic Finite Automaton): efisien untuk eksekusi. Dibuat dari NFA menggunakan subset construction (algoritma determinisasi).
- Minimasi DFA (contoh: Hopcroft) mengurangi jumlah state untuk tabel transisi yang lebih ringkas.

Proses praktis dalam lexer generator:

1. Tulis sejumlah ekspresi reguler untuk setiap token.
2. Buat NFA untuk setiap ekspresi reguler.
3. Gabungkan NFA menjadi satu mesin dengan ε-transisi dari start baru.
4. Konversi gabungan NFA → DFA.
5. Minimalkan DFA dan hasilkan tabel transisi / kode sumber.

## 5. Aturan Pengenalan Token (Longest Match & Priority)

Konflik dapat muncul ketika sebuah substring dapat cocok dengan lebih dari satu pola (mis. `if` sebagai IDENTIFIER dan juga sebagai keyword `if`). Dua aturan umum:

- Longest match (maximal munch): pilih kecocokan terpanjang.
- Priority (precedence): jika dua pola sama panjang, gunakan prioritas yang didefinisikan (biasanya urutan aturan dalam file `.l` atau tabel prioritas).

Contoh: aturan `=` dan `==` → input `==` harus dikenali sebagai `==` (operator relasi), bukan dua token `=`.

## 6. Arsitektur Lexer

Komponen utama:
- Input buffer (two-buffer scheme) untuk efisiensi baca file.
- Pointer `lexeme_begin` dan `forward` untuk menandai batas lexeme aktif.
- Mesin keadaan (DFA) yang menavigasi karakter demi karakter.
- Tabel aksi untuk state akhir (accepting states) yang menghasilkan token dan atribut.

Buffering: teknik dua-buffer (setiap buffer mis. 4KB) memungkinkan rollback kecil dan deteksi EOF tanpa overhead per-character I/O.

## 7. Penanganan Spesial

- Whitespace & komentar: biasanya diabaikan di lexer (token di-drop).
- Reserved words (keyword) vs identifier: setelah mengenali IDENTIFIER, cek di tabel keyword — jika ada, ubah tipe token menjadi keyword.
- String literal: perlu penanganan escape (`\\n`, `\\"`) dan deteksi unterminated string.
- Number formats: integer vs float — urutan regex penting untuk membedakan `123` dan `123.45`.

## 8. Error Leksikal

Jenis error:
- Invalid character: simbol yang tidak ada dalam alphabet bahasa.
- Unterminated string/comment.
- Numeric overflow pada parsing atribut.

Strategi penanganan:
- Laporkan posisi baris/kolom dan lexeme bermasalah.
- Recovery sederhana: skip satu karakter, atau skip sampai delimiter aman (`;` atau newline), catat error dan lanjut.

## 9. Implementasi: Contoh Aturan Token

Berikut contoh himpunan token dasar untuk bahasa mirip C:

- KEYWORD: `if|else|while|for|return|int|float|char|void`
- IDENTIFIER: `[A-Za-z_][A-Za-z0-9_]*`
- INTEGER_LITERAL: `[0-9]+`
- FLOAT_LITERAL: `([0-9]+)\\.[0-9]+([eE][+-]?[0-9]+)?`
- STRING_LITERAL: `"(\\\\.|[^"\\\\])*"`
- OPERATOR: `\\+|-|\\*|/|==|!=|<=|>=|<|>|=`
- SEPARATOR: `[(){};,]`
- COMMENT: `//.*|/\\*(.|\\n)*?\\*/` (hapus/abaikan)
- WHITESPACE: `[ \\t\\r\\n]+` (abaikan)

## 10. Contoh File Flex (flex / lex)

Contoh sederhana `mini.l`:

```lex
%{
#include "tokens.h"
%}

%%
[ \t\r\n]+            /* ignore whitespace */
//.*                  /* ignore line comments */
/\*([^*]|\*+[^*/])*\*+/  /* ignore block comments */
"(\\.|[^"\\])*"    { yylval.str = strdup(yytext); return STRING_LITERAL; }
[0-9]+\.[0-9]+([eE][+-]?[0-9]+)? { yylval.fval = atof(yytext); return FLOAT_LITERAL; }
[0-9]+                { yylval.ival = atoi(yytext); return INTEGER_LITERAL; }
[A-Za-z_][A-Za-z0-9_]* { if (is_keyword(yytext)) return KEYWORD; else { yylval.id = strdup(yytext); return IDENTIFIER; } }
==|!=|<=|>=|\+|-|\*|/|<|>|= { return OPERATOR; }
[(){};,]              { return SEPARATOR; }
.                     { fprintf(stderr, "Unknown char: %s\\n", yytext); }
%%

int main(int argc, char **argv) {
    yylex();
    return 0;
}
```

Catatan: tool `flex` menghasilkan `lex.yy.c`, kemudian kompilasi dengan `gcc lex.yy.c -lfl`.

## 11. Contoh Lexer Hand-Written (Python, regex-based)

Berikut contoh pendek menggunakan `re` di Python untuk mengilustrasikan konsep:

```python
import re

token_specification = [
    ('NUMBER',   r'\\d+(?:\\.\\d*)?'),
    ('ID',       r'[A-Za-z_]\\w*'),
    ('STRING',   r'"(?:\\\\.|[^"\\\\])*"'),
    ('OP',       r'==|!=|<=|>=|[+\\-*/=<>]'),
    ('SKIP',     r'[ \\t]+'),
    ('NEWLINE',  r'\\n'),
    ('MISMATCH', r'.'),
]

tok_regex = '|'.join('(?P<%s>%s)' % pair for pair in token_specification)
get_token = re.compile(tok_regex).match

def tokenize(code):
    pos = 0
    line = 1
    mo = get_token(code, pos)
    while mo:
        kind = mo.lastgroup
        value = mo.group()
        if kind == 'NEWLINE':
            line += 1
        elif kind != 'SKIP':
            yield (kind, value, line)
        pos = mo.end()
        mo = get_token(code, pos)
    if pos != len(code):
        raise RuntimeError('Unexpected character %r on line %d' % (code[pos], line))

# Example
code = 'x = 3 + 42.5\\nprint("hello")'
print(list(tokenize(code)))
```

Pendekatan ini sederhana, tetapi untuk bahasa produksi biasanya dibutuhkan DFA yang dihasilkan oleh generator untuk efisiensi.

## 12. Contoh Implementasi Hand-Coded (C++ sederhana)

Contoh berikut menunjukkan pendekatan state-machine sederhana untuk mengenali identifier dan integer saja:

```cpp
#include <iostream>
#include <string>
#include <cctype>

enum TokenType { T_ID, T_INT, T_PLUS, T_EOF, T_UNKNOWN };

struct Token { TokenType type; std::string lexeme; };

class Lexer {
    std::string s; size_t i = 0;
public:
    Lexer(const std::string &src): s(src) {}
    Token next() {
        while (i < s.size() && isspace((unsigned char)s[i])) ++i;
        if (i >= s.size()) return {T_EOF, ""};
        char c = s[i];
        if (isalpha((unsigned char)c) || c == '_') {
            size_t start = i;
            while (i < s.size() && (isalnum((unsigned char)s[i]) || s[i]=='_')) ++i;
            return {T_ID, s.substr(start, i-start)};
        }
        if (isdigit((unsigned char)c)) {
            size_t start = i;
            while (i < s.size() && isdigit((unsigned char)s[i])) ++i;
            return {T_INT, s.substr(start, i-start)};
        }
        if (c == '+') { ++i; return {T_PLUS, "+"}; }
        ++i; return {T_UNKNOWN, std::string(1,c)};
    }
};

int main() {
    Lexer lx("x + 123 var_name");
    for (;;) {
        Token t = lx.next();
        if (t.type == T_EOF) break;
        std::cout << "Token: " << t.lexeme << "\n";
    }
}
```

Kode ini mengilustrasikan ide dasar: scan karakter dan beralih berdasarkan kelas karakter.

## 13. Integrasi dengan Parser

Interface umum: lexer memberikan token ke parser melalui fungsi seperti `yylex()` yang mengembalikan kode token dan mengisi `yylval` dengan atribut. Parser (mis. YACC/Bison) mengonsumsi token ini.

Poin penting:
- Token harus membawa informasi posisi (baris, kolom) untuk pesan error.
- Lexer tidak boleh melakukan kerja parsing (mis. memeriksa konteks yang membutuhkan struktur pohon).

## 14. Unicode dan Encoding

Untuk bahasa modern, perhatikan encoding sumber (UTF-8 umum). Tantangan:
- Identifier yang mengizinkan karakter Unicode: perlu klasifikasi karakter berdasarkan kategori Unicode (Letter, Number, ConnectorPunctuation).
- Sering lebih mudah untuk membaca byte UTF-8 dan menerjemahkannya ke code points sebelum DFA.

## 15. Optimisasi

- Gabungkan semua regex token menjadi satu DFA untuk menghindari backtracking dan pemindaian multi-pola.
- Gunakan tabel transisi (array) untuk cepat mengakses state berikutnya.
- Kompresi tabel, teknik perfect hashing untuk class character (mis. ASCII vs Unicode).
- JIT code generation: generator lexer yang langsung menghasilkan switch/case teroptimasi.

## 16. Latihan dan Tugas

1. Tulis file `mini.l` (flex) untuk bahasa kecil yang mengenali angka, identifier, operator, komentar, string. Compile dan jalankan.
2. Implementasikan lexer hand-written untuk bahasa ekspresi aritmetika dan uji dengan input kompleks.
3. Ambil regex untuk identifier dan integer, buat NFA, lalu konversi menjadi DFA (manual untuk contoh kecil).
4. Bandingkan performa scanner berbasis regex (Python) vs scanner hasil `flex` pada file sumber 100 KB.

## 17. Ringkasan

Analisis leksikal adalah komponen esensial yang menjembatani input karakter dengan parser. Dengan pengetahuan tentang ekspresi reguler, automata hingga, dan teknik implementasi (hand-coded atau generator seperti `flex`), Anda dapat merancang lexer yang akurat dan efisien.

---

## Referensi Singkat

- Aho, Lam, Sethi, Ullman — "Compilers: Principles, Techniques, and Tools" (Dragon Book)
- John Levine — "Flex & Bison"
- Dokumentasi `flex`, `re2c`, `ragel`, `ANTLR` untuk praktik lexer generator
