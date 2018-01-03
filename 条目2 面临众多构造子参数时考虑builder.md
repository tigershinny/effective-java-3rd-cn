静态工厂和构造子都有一个限制：它们不能很好调节大量可选参数。考虑这个情况，一个类表示包装食品上的营养成分标签。这些标签有些是必需的域，食用份量、份数和每份卡路里数；还有二十多个可选域，总脂肪含量、饱和脂肪含量、反脂肪含量、胆固醇含量，钠含量等等。大多数产品中这些可选参数只有很少的非零值。

对于这样的类，你打算用什么类型的构造子或者静态工厂？习惯上，编程人员用*重叠构造子(telescoping constructor)模式*：提供仅仅必需参数的构造子，有单个可选参数的另外一个构造子，和有两个可选参数的第三个构造子，等等，最后有所有可选参数的构造子。这是实践中的样子。为了简单起见，只展示四个可选参数：

```java
// 重叠构造子模式 - 扩展性不好!
public class NutritionFacts {
    private final int servingSize;  // (mL)            必需的
    private final int servings;     // (per container) 必需的
    private final int calories;     // (per serving)   可选的
    private final int fat;          // (g/serving)     可选的
    private final int sodium;       // (mg/serving)    可选的
    private final int carbohydrate; // (g/serving)     可选的

    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }
    public NutritionFacts(int servingSize, int servings,
            int calories) {
        this(servingSize, servings, calories, 0);
    }

    public NutritionFacts(int servingSize, int servings,
            int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
    }

    public NutritionFacts(int servingSize, int servings,
            int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat, sodium, 0);
    }

    public NutritionFacts(int servingSize, int servings,
           int calories, int fat, int sodium, int carbohydrate) {
        this.servingSize  = servingSize;
        this.servings     = servings;
        this.calories     = calories;
        this.fat          = fat;
        this.sodium       = sodium;
        this.carbohydrate = carbohydrate;
    }
}
```
当你想创建一个实例，用包含你想设置所有参数的最短参数列表的构造子：

```java
NutritionFacts cocaCola = new NutritionFacts(240, 8, 100, 0, 35, 27);
```
通常，调用这样的构造子需要你不想设置的参数，但是你被迫为它们传递参数。这种情况下，你传递0这个值给fat。对于“只有”六个参数的情况，这个看上去不算很糟，但是当参数数量增加时，很快就会失控。

简而言之，**重叠构造子模式奏效，但是当有很多参数的时候，写客户端代码是困难的，读代码更困难**。阅读的人会想，这些值是什么，而且必须细心计算参数来弄明白。相同类型参数的长长的系列会造成微妙的bug。如果客户端不慎颠倒了两个这样的参数，编译器不会发现，但是程序在运行时异常(条目51)。

当面临许多可选参数的构造子时，第二个替代方案是JavaBean模式，这个模式中，你调用一个无参数的构造子来创建一个对象，然后调用设置方法来设置每个必需的参数和感兴趣的可选参数：
```java
// JavaBean模式 - 允许不一致性, 支持可变性
public class NutritionFacts {
    // 初始化为默认值的参数(如果有)
    private int servingSize  = -1; // 必需的; 无默认值
    private int servings     = -1; // 必需的; 无默认值
    private int calories     = 0;
    private int fat          = 0;
    private int sodium       = 0;
    private int carbohydrate = 0;

    public NutritionFacts() { }
    // Setters
    public void setServingSize(int val)  { servingSize = val; }
    public void setServings(int val)    { servings = val; }
    public void setCalories(int val)    { calories = val; }
    public void setFat(int val)         { fat = val; }
    public void setSodium(int val)      { sodium = val; }
    public void setCarbohydrate(int val) { carbohydrate = val; }
}
```
这个模式没有重叠构造子模式的缺点。虽然有点啰嗦，但是容易创建实例，代码也容易阅读：

```java
NutritionFacts cocaCola = new NutritionFacts();
cocaCola.setServingSize(240);
cocaCola.setServings(8);
cocaCola.setCalories(100);
cocaCola.setSodium(35);
cocaCola.setCarbohydrate(27);
```
不幸的是，JavaBean模式有它自己的严重缺点。因为构造被分成多个调用，**一个JavaBean在构造的过程中有可能不一致的状态**。这个类没有这个选择：检查构造子参数的有效性来实行一致性。尝试使用一个不一致状态的对象，会导致失败，这个失败与含有bug的代码大相径庭，所以难于调试。一个相应的缺点是，**JavaBean模式限制了生成一个不变类的可能性**，需要编码者花额外的功夫保证线性安全。
手动“冻结(freezing)”对象来减少这些缺点是可能的：当对象构造完成了，不允许使用直到冻结。但是这个变体是笨拙的，在实践中用的非常稀少。此外，在运行中会造成错误，因为在使用它之前，编译器不能保证编码者调用一个对象的冻结方法。
幸运的是，有第三种选择，可以结合重叠构造子模式的安全性和JavaBean模式的可读性。这个是*Builder*模式[Gamma95]的一种形式。不是直接生成对象，而是客户端调用有所有必需参数的一个构造子(或者静态工厂)，获得一个*builder*对象。然后客户端在builder对象上调用类似设置方法，分别设置感兴趣的可选参数。最后，客户端调用无参数的build方法来生成对象，这个对象一般是不可变的。builder是一个它构建的类的静态成员类(条目24)。下面是在实践中的样子：

```java
// Builder模式
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        // 必需的参数
        private final int servingSize;
        private final int servings;

        // 可选的参数 - 初始化为默认值
        private int calories      = 0;
        private int fat           = 0;
        private int sodium        = 0;
        private int carbohydrate  = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings    = servings;
        }

        public Builder calories(int val)
            { calories = val;      return this; }
        public Builder fat(int val)
            { fat = val;           return this; }
        public Builder sodium(int val)
            { sodium = val;        return this; }
        public Builder carbohydrate(int val)
            { carbohydrate = val;  return this; }
        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize  = builder.servingSize;
        servings     = builder.servings;
        calories     = builder.calories;
        fat          = builder.fat;
        sodium       = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```
NutritionFacts类是不变的，所有参数的默认值在一个地方。builder的设置方法返回builder自身，可以链式调用，所以有流畅的API。下面是客户端代码的样子：

```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
        .calories(100).sodium(35).carbohydrate(27).build();
```
客户端代码容易写，更重要的是也容易通读。**builder模式模仿了在Python和Scala中有名字的可选参数**。

为了简单，省略了有效性检查。为了尽快发现无效的参数，在builder构造子和方法中检查参数的有效性。build方法调用的有多个参数的构造子，检查这些不变量。为了保证这些不变量不受攻击，在拷贝builder的参数后(条目50)，务必对对象的域进行检查。如果检查失败，抛出IllegalArgumentException，它的具体信息表示哪些参数是无效的(条目75)。

**builder模式非常适合类继承**。使用并行的builder的层级，每个builder嵌套在对应的类中。抽象的类有抽象的builder；具体的类有具体的builder。比如，考虑一个抽象类是代表各种种类披萨的层级的根类:
```java
// 为类层级的Builder模式
public abstract class Pizza {
   public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE }
   final Set<Topping> toppings;

   abstract static class Builder<T extends Builder<T>> {
      EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);
      public T addTopping(Topping topping) {
         toppings.add(Objects.requireNonNull(topping));
         return self();
      }

      abstract Pizza build();

      // Subclasses must override this method to return "this"
      protected abstract T self();
   }
   Pizza(Builder<?> builder) {
      toppings = builder.toppings.clone(); // See Item  50
   }
}
```
注意，Pizza.Builder是一个有*递归类型参数(recursive type parameter)*(条目30)的*泛型(generic type)*。这个和抽象的self方法一起，可以在子类中使得链式方法奏效，而不需要强行转换。Java缺少self类型，这个变通方案被认为是模拟self类型的惯例。

Pizza有两个具体的子类，一个是纽约类型的比萨，另外一个是半月比萨。前者有指定大小的参数，而后者需要指定酱汁是在外面还是里面。

```java
public class NyPizza extends Pizza {
    public enum Size { SMALL, MEDIUM, LARGE }
    private final Size size;

    public static class Builder extends Pizza.Builder<Builder> {
        private final Size size;

        public Builder(Size size) {
            this.size = Objects.requireNonNull(size);
        }

        @Override public NyPizza build() {
            return new NyPizza(this);
        }

        @Override protected Builder self() { return this; }
    }

    private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
    }
}

public class Calzone extends Pizza {
    private final boolean sauceInside;

    public static class Builder extends Pizza.Builder<Builder> {
        private boolean sauceInside = false; // 默认

        public Builder sauceInside() {
            sauceInside = true;
            return this;
        }

        @Override public Calzone build() {
            return new Calzone(this);
        }

        @Override protected Builder self() { return this; }
    }

    private Calzone(Builder builder) {
        super(builder);
        sauceInside = builder.sauceInside;
    }
}
```
注意， 每个子类builder中的build方法是声明返回正确的子类的：NyPizza.Builderde的build方法返回NyPizza，而Calzone.Builder的build方法返回Calzone。这个技巧，一个子类方法被声明返回一个声明在超类中返回类型的子类型，叫做*协变返回类型(covariant return typing)*。这些层级builder的客户端代码，本质上与简单的NutritionFacts builder代码是等同的。

客户端的代码例子如下，为了简洁，假设enum常量是静态导入：

```java
NyPizza pizza = new NyPizza.Builder(SMALL)
        .addTopping(SAUSAGE).addTopping(ONION).build();
Calzone calzone = new Calzone.Builder()
        .addTopping(HAM).sauceInside().build();
```
相对于构造子，builder有个小优点是，builder可以有多个可变参数，因为每个参数是它自己方法中指定的。或者，build可以把传入到多次调用的方法的参数聚合到单个域，就像前面addTopping方法展示的一样。

builder模式相当灵活。单个builder可以重复使用到构建多个对象。随着创建对象的不同，builder的参数在build方法的调用之间可以改变。对象创建时builder可以自动填充到域中，比如随着每次创建时一个增加的系列数。

builder模式也有缺点，为了创建一个对象，必须首先创建它的builder。创建builder的代价在实践中虽然是不显而易见，但是在性能要求苛刻的情形中是一个问题。而且，builder模式相对于重叠构造子模式更加啰嗦，所以如果有足够的参数，比如四个或者更多，才值得使用。但是记住，未来你有可能添加更多的参数。但是，如果你开始使用构造子或者静态工厂，然后当类演变到参数个数失控的时候，切换到builder，这时废弃的构造子或者静态工厂一直存在，这是个尴尬的事情。

总之，**在设计类时，当它的构造子或者静态工厂有多过一些参数的时候，builder模式是一个好的选择，特别是当多个参数是可选的或者是同个类型的时候**。和重叠构造子相比，客户端代码更容易阅读和编写，而且builder比JavaBeans更安全。
