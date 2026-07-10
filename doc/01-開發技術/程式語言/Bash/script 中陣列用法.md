1. **定義或是宣告**

```bash
ethernet_test_subitems=($(seq -f "Port%.f" 1 2))
declare -a ethernet_result=("FAIL" "FAIL")
declare -a interface_info=()
```

2. **取的陣列內容**

```bash
#取出第一筆
local value=${info[1]}

#得到當前陣列個數
local count=${#info[@]}
```

3. **取得所有陣列內容**

@ 展開陣列會當作多個參數輸入

```bash
# 使用 @

array=("a b" "c" "d")
echo ${array[@]}
# 輸出：a b c d

#加引號
echo "${array[@]}"
# 輸出："a b" "c" "d"
```

＊展開陣列會全部當作整合成一個參數輸入

```bash
#使用 *

array=("a b" "c" "d")
echo ${array[*]}
# 輸出：a b c d

#加引號，並且變更分隔號為,
IFS=","
echo "${array[*]}"
# 輸出：a b,c,d
```

