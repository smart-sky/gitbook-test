###获取某字符串前几位
    
```java 
public static void main(String[] args) {
        String s = "asdijaoisjd";
        //该方法根据下标，包头不包尾
        String s1 = s.substring(0,5);
        String s2 = s.substring(1,5);
        //超出数组下标会报出错误
        //String s3 = s.substring(0,20);
        System.out.println(s);
        System.out.println(s1);
        System.out.println(s2);
       // System.out.println(s3);
    }

```
