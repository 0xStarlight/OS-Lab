
```bash
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.5.tar.xz
```

![[Pasted image 20230830142655.png]]

```bash
ls 
tar xvf linux-6.5.tar.xz
```

![[Pasted image 20230830142937.png]]

```bash
sudo apt update
sudo apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison
```

![[Pasted image 20230830143546.png]]

```bash
cd linux-6.5
cp -v /boot/config-$(uname -r) .config
make menuconfig
```

![[Pasted image 20230830143935.png]]


![[Pasted image 20230830143752.png]]


![[Pasted image 20230830153428.png]]

![[Pasted image 20230830153847.png]]

![[Pasted image 20230830153956.png]]

![[Pasted image 20230830154719.png]]

![[Pasted image 20230830154751.png]]

![[Pasted image 20230830154825.png]]

![[Pasted image 20230830155722.png]]

![[Pasted image 20230830155844.png]]

![[Pasted image 20230901222543.png]]








