# Questions
#### 1. Here is the SPL Code to print odd numbers from 1 to 20 :
```python
alias counter R0;
counter = 0;
while(counter <= 20) do
  if(counter%2 != 0) then
    print counter;
  endif;
  counter = counter + 1;
endwhile;
```

#### 2. Sum from 1 to 20
```python
alias counter R0;
alias sum R1;
counter = 0;
sum = 0;
while(counter <= 20) do
	sum = sum + counter;
	counter = counter + 1;
endwhile;
print sum;
```

#### 3. Print sum of squares of the first 20 natural numbers.
```python
alias counter R0;
alias sum R1;
alias tmp R2;
counter = 0;
sum = 0;
tmp = 1;
while(counter <= 20) do
	tmp = counter * counter;	
	sum = sum + tmp;
	counter = counter + 1;
endwhile;
print sum;
```