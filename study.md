1、拆箱、装箱处理

````java
	public static void main(String[] args) {
        Integer i1 = 40;
        Integer i2 = 40;
        Integer i3 = 0;
        Integer i4 = new Integer(40);
        Integer i5 = new Integer(40);
        Integer i6 = new Integer(0);

        // true,两个Integer比较， 在[-128, 127]内相等，否则不相等，因为会重新new一个对象；
        System.out.println("i1=i2   " + (i1 == i2));
        // ture,i2+i3会拆箱为int类型，因此是数值比较
        System.out.println("i1=i2+i3   " + (i1 == i2 + i3));
        // false,两个对象做比较，比较的是两个引用指向的对象地址
        System.out.println("i1=i4   " + (i1 == i4));
        // false，同上
        System.out.println("i4=i5   " + (i4 == i5));
        // ture, i4 == i5 + i6，因为+这个操作符不适用于 Integer 对象，首先 i5 和 i6 进行自动拆箱操作，进行数值相加，即 i4 == 40。然后 Integer 对象无法与数值进行直接比较，所以 i4 自动拆箱转为 int 值 40，最终这条语句转为 40 == 40 进行数值比较
        System.out.println("i4=i5+i6   " + (i4 == i5 + i6));
        // true，数值比较
        System.out.println("40=i5+i6   " + (40 == i5 + i6));
    }
````

