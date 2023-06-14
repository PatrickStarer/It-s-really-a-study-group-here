* java中的String类，
* String str1 = "hello World"; 这个str1放在jvm的哪个部分？
* String str2 = new String("Hello World");这个str2放在jvm的哪个部分？

```java

        String str1 = "s1";
        String str2 = new String("s1");
        System.out.println(str1 == str2);
        //true or false?

        String str1 = "s1";
        String str2 = new String("s1");
        String str3 = str2.intern();
        System.out.println(str1 == str3);
         //true or false ? 这里intern的作用是什么



```