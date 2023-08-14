## Find square of first 10 numbers

```nasm
check:
MOV R0, 1
MOV R3, 1
MOV R1, 10

find_square:
MUL R0, R0
PORT P1, R0
OUT
INR R3
MOV R0,R3
DCR R1
JZ R1, exit
JMP find_square

exit:
HALT
```

