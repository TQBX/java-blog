#### [剑指 Offer 20. 表示数值的字符串](https://leetcode-cn.com/problems/biao-shi-shu-zhi-de-zi-fu-chuan-lcof/)

请实现一个函数用来判断字符串是否表示数值（包括整数和小数）。例如，字符串"+100"、"5e2"、"-123"、"3.1416"、"-1E-16"、"0123"都表示数值，但"12e"、"1a3.14"、"1.2.3"、"+-5"及"12e+5.4"都不是。

```java
class Solution {
    public boolean isNumber(String s) {
        if(s == null || s.length() == 0) return false;
        boolean isNum = false,isDot = false,isE = false;

        char[] str = s.trim().toCharArray();

        for(int i = 0; i < str.length; i++){
            if(str[i] >= '0' && str[i] <= '9') isNum = true;
            else if(str[i] == '.'){
                //if(!isNum) return false;  我以为.1不算是有效数字
                if(isDot || isE) return false;//小数点之前出现过e或者小数点，表示错误 .e
                isDot = true;
            }
            else if(str[i] == 'e' || str[i] == 'E'){
                 if(!isNum || isE) return false;//e或E之前必须出现数字且不能重复出现e
                 isE = true;
                 isNum = false; //重置isNum，排除123e或者123e+
            }
            else if(str[i] == '+' || str[i] == '-'){
                if(i != 0 && str[i - 1] != 'e' && str[i-1]!= 'E') return false;//+ - 只能出现在第一个位置，或者e的后面
            }
            else return false;
        }

        return isNum;
    }
}
```

