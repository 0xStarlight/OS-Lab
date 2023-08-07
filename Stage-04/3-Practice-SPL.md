# Questions
#### 1. Here is the SPL Code to print odd numbers from 1 to 20 :
```nasm
alias counter R0;
counter = 0;
while(counter <= 20) do
  if(counter%2 != 0) then
    print counter;
  endif;
  counter = counter + 1;
endwhile;
```