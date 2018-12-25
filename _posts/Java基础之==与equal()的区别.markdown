## ==

== 是Java中的运算符。对于基础数据类型是比较的是两个变量的值是否相等，对于引用型变量表示的是两个变量在堆中存储的地址是否相同，即栈中的内容是否相同。

## equal()

equal()是object类中的方法，是用来比较两个对象内部的内容是否相等的，由于所有的类都是继承自java.lang.Object类的，所以如果没有对该方法进行重写的话，调用的仍然是Object类中的方法，而Object中的equal方法返回的却是==的判断，因此，如果在没有进行该方法的覆盖后，调用该方法就和==一样。在java面向对象的处理中我们一般在javabean中都要选择重写equals方法，使用hibernate后，我们要生成数据库的映射文件与实体类，这是我们就最好在实体类中进行equals方法的重写，重写时我们可以根据自己的定义来实现该方法只要遵守那五条原则

>1、自反性：对任意引用值X，x.equals(x)的返回值一定为true.
>2、对称性：对于任何引用值x,y,当且仅当y.equals(x)返回值为true时，x.equals(y)的返回值一定为true;
>3、传递性：如果x.equals(y)=true, y.equals(z)=true,则x.equals(z)=true
>4、一致性：如果参与比较的对象没任何改变，则对象比较的结果也不应该有任何改变
>5、非空性：任何非空的引用值X，x.equals(null)的返回值一定为false
>String重写了equal()方法，比较的不再是引用，而是比较的值是否相同
>如果没重写equal()方法的话，equal()和==一样都是比较在堆中存储的地址是否相同，重写equal()大部分都是比较是否是对同一对象的引用。
>对于String ,基本类型的包装类型Boolean、Character、Byte、Shot、Integer、Long、Float、Double。equal()表示比较内容 。
>一般api中继承object的类都已重写equal对内容进行比较。