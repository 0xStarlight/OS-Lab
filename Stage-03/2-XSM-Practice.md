# Extra Questions
## Q1. Multiplies of 3 till 20

```nasm
check:
mov R0, 20
mov R1, 1

print_numbers:
port P1, R1
OUT
inr R1
inr R1
inr R1
dcr R0
JZ R0,exit
JMP print_numbers

exit:
HALT
```

```nasm
check:
mov R0, 20
mov R1, 3
mov R2, 3

print_numbers:
port P1, R1
OUT
ADD R1, R2
dcr R0
JZ R0,exit
JMP print_numbers

exit:
HALT
```

