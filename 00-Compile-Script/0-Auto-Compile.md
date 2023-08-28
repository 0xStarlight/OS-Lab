# Work in progress
### Todo
1. [ ] Auto load script

---
# SPL Script
> Save in /spl dir
```bash
# Define color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

if [ $# -ne 1 ]; then
    echo -e "${RED}Usage: $0 <stage_name>${NC}"
    exit 1
fi

dir_name="$1"

input_spl_dir="$HOME/myexpos/spl/spl_progs/$dir_name"

echo -e "${GREEN}[+] Compiling SPL files...${NC}"

if [ ! -d "$input_spl_dir" ]; then
    echo -e "${RED}[-] No SPL files found to compile"${NC}
    exit 1
fi

# Compile individual SPL files
for file in "$input_spl_dir"/*.spl; do
    echo -e "${NC}[+] Compiling $file${NC}"
    filename=$(basename "$file")
    "$HOME/myexpos/spl/spl" "$file"
done
echo -e "${GREEN}[+] Compiled all SPL files${NC}"
```

---

# EXPL Script
> Save in /expl dir
```bash
# Define color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

if [ $# -ne 1 ]; then
    echo -e "${RED}Usage: $0 <stage_name>${NC}"
    exit 1
fi

dir_name="$1"

input_expl_dir="$HOME/myexpos/expl/expl_progs/$dir_name"

echo -e "${GREEN}[+] Compiling EXPL files...${NC}"

# Check if the input directory exists
if [ ! -d "$input_expl_dir" ]; then
    echo -e "${RED}[-] No EXPL files found to compile${NC}"
    exit 1
fi

# Compile individual EXPL files

for file in "$input_expl_dir"/*.expl; do
    echo -e "${NC}[+] Compiling $file"
    filename=$(basename "$file")
    "$HOME/myexpos/expl/expl" "$file"
done
echo -e "${GREEN}[+] Compiled all EXPL files${NC}"
```

---

# Master Script
> Save in /test dir
```bash
# Define color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

if [ $# -ne 1 ]; then
    echo -e "${RED}Usage: $0 <stage_name>${NC}"
    exit 1
fi

stage_name="$1"

compile_spl="$HOME/myexpos/spl"
compile_expl="$HOME/myexpos/expl"
home="$HOME/myexpos/test"


# Compile all the files
echo "${GREEN}[+] Compiling all the required files...${NC}"
echo
cd $HOME/myexpos/spl
bash compile.sh $stage_name
cd $HOME/myexpos/expl
echo
bash compile.sh $stage_name
cd $HOME/myexpos/test
echo

echo "${GREEN}[+] Compilation complete.${NC}"
```

---

# How to use
> Save all the spl files for the current stage in spl_progs/stage+number
> Do same for expl

```bash
$> ./compile stage12
```

![[Pasted image 20230828222937.png]]




























