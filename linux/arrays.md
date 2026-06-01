```bash
declare -a fruits=("apple" "banana" "cherry")
echo ${fruits[0]}   # apple
echo ${fruits[1]}   # banana

fruits+=("date")  # add an element to the array
echo ${fruits[@]}  # apple banana cherry date
echo ${!fruits[@]}  # 0 1 2 3

# declare -a NODES=("a" "b" "c")
# echo ${NODES[@]}
# declare -a EMPTY
echo
echo
echo
echo "nodes-------------"
echo
declare -A NODES
NODES=(
  [control-plane]="10.0.4.229"
  [worker-1]="10.0.15.20"
  [worker-2]="10.0.5.214"
  [worker-3]="10.0.4.240"
)

# IFS custom separator for arrays
IFS=','
# always reset IFS to default after use, otherwise it can cause issues with word splitting in loops and other commands
IFS=$' \t\n'
  
echo "keys   ${NODES[@]}"
echo "values ${!NODES[@]}"
echo "count  ${#NODES[@]}"
  
echo
echo
echo "Iterating"
 
for NODE in ${!NODES[@]}; do
   IPP="${NODES[$NODE]}"
   echo "${NODE} ip is ${IPP}"
done
```