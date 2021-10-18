---
layout: post
title: 组合优于继承
date: 2018-07-11 09:51
tags: Java Go Kotlin 组合
categories: 编程
---

《Effective Java 中文版第2版》书中第16条中说到：

> 继承是实现代码复用的有力手段，但它并非永远是完成这项工作的的最佳工具。

## 继承有什么问题？

继承打破了类的封装性，子类依赖于父类中特定功能的实现细节。

## 继承什么时候是安全的

- 在包的内部是用继承，不存在跨包继承。
- 专门为了扩展而设计，并且具备很好的文档说明。

## 一个例子

实现这样一个`HashSet`，可以跟踪从它被创建之后曾经添加过几个元素。

### 使用继承实现

```java
public class InstrumentedSet<E> extends HashSet<E> {
  // The number of attempted element insertions
  private int addCount = 0;

  public InstrumentedSet() {
  }

  public InstrumentedSet(int initCap, float loadFactor) {
    super(initCap, loadFactor);
  }

  @Override
  public boolean add(E e) {
    addCount++;
    return super.add(e);
  }

  @Override
  public boolean addAll(Collection<? extends E> c) {
    addCount += c.size();
    return super.addAll(c);
  }

  public int getAddCount() {
    return addCount;
  }
}
```

类中使用 `addCount` 字段记录添加元素的次数，并覆盖父类的 `add()`和 `addAll()` 实现，对 `addCount` 字段进行设值。

在下面的程序中，我们期望 `getAddCount()` 返回3，但实际上返回的是6。

```java
InstrumentedSet<String> s = new InstrumentedSet<String>();
s.addAll(Arrays.asList("Snap", "Crackle", "Pop"));
```

问题出在于：在 `HashSet` 中，`addAll()` 的实现是基于 `add()` 方法的。子类在扩展父类的功能时，如果不清楚实现细节，是非常危险的，况且父类的实现在未来可能是变化的，毕竟它并不是为扩展而设计的。

### 使用组合实现

不用扩展现有的类，而是在新的类中增加一个私有字段，引用现有类的实例。这种设计被叫做**组合**。

先创建一个干净的 `SetWrapper` 组合类。

```java
public class SetWrapper<E> implements Set<E> {
  private final Set<E> s;
  public SetWrapper(Set<E> s) { this.s = s; }
  public void clear()               { s.clear();            }
  public boolean contains(Object o) { return s.contains(o); }
  public boolean isEmpty() { return s.isEmpty();   }
  public int size() { return s.size();      }
  public Iterator<E> iterator() { return s.iterator();  }
  public boolean add(E e) { return s.add(e);      }
  public boolean remove(Object o){ return s.remove(o);   }
  public boolean containsAll(Collection<?> c) { return s.containsAll(c); }
  public boolean addAll(Collection<? extends E> c) { return s.addAll(c);      }
  public boolean removeAll(Collection<?> c) { return s.removeAll(c);   }
  public boolean retainAll(Collection<?> c) { return s.retainAll(c);   }
  public Object[] toArray()          { return s.toArray();  }
  public <T> T[] toArray(T[] a)      { return s.toArray(a); }
  @Override public boolean equals(Object o) { return s.equals(o);  }
  @Override public int hashCode()    { return s.hashCode(); }
  @Override public String toString() { return s.toString(); }
}
```

`SetWrapper` 实现了装饰模式，通过引用 `Set<E>` 类型的字段，面向接口编程，相比直接继承 `HashSet` 类来得更灵活。可以在调用该类的构造方法中传入任意 `Set` 具体类。扩展该类以实现需求。

```java
public class InstrumentedSet<E> extends SetWrapper<E> {
  private int addCount = 0;

  public InstrumentedSet(Set<E> s) {
    super(s);
  }

  @Override
  public boolean add(E e) {
    addCount++;
    return super.add(e);
  }

  @Override
  public boolean addAll(Collection<? extends E> c) {
    addCount += c.size();
    return super.addAll(c);
  }

  public int getAddCount() {
    return addCount;
  }
}
```

## 举一反三

**注：以下代码均是伪代码，组合方式的实现封装成 Android 库并已开源，名叫 [Modapter](https://github.com/samelody/modapter)（ Modular Adapter 之意）。没错，这里打了个广告。**

先来看看问题。笔者曾开发的某个应用有以下2张截图：

![游戏详情页面](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/11/15/1671506a97e8bc39~tplv-t2oaga2asx-image.image)

![评论列表页面](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/11/15/1671506a97fec3a8~tplv-t2oaga2asx-image.image)

详情页面和评论列表页面均复用了评论项的实现。

评论列表页面的  `GameComentsAdapter`。

```java
public class GameCommentsAdapter extends RecyclerView.Adapter<BaseViewHolder> {
    private static final int ITEM_TYPE_COMMENT = 1;
    private List<Object> mDataSet;

    @Override
    public int getItemViewType(int position) {
        Object item = getItem(position);
        if (item instanceof Comment) {
            return ITEM_TYPE_COMMENT;
        }
        return super.getItemViewType(position);
    }

    protected Object getItem(int position) {
        return mDataSet.get(position);
    }

    @NonNull
    @Override
    public BaseViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        LayoutInflater inflater = LayoutInflater.from(parent.getContext());
        if (viewType == ITEM_TYPE_COMMENT) {
            View itemView = inflater.inflate(R.layout.item_comment, parent, false);
            return new CommentViewHolder(itemView);
        }
        return null;
    }

    @Override
    public int getItemCount() {
        return mDataSet.size();
    }
}
```

### if-else 方式实现

修改 `GameComentsAdapter` 类，增加对游戏详情项的适配支持。

```java
public class GameCommentsAdapter extends RecyclerView.Adapter<BaseViewHolder> {
    private static final int ITEM_TYPE_COMMENT = 1;
    private static final int ITEM_TYPE_GAME_DETAIL = 2;
    private List<Object> mDataSet;

    @Override
    public int getItemViewType(int position) {
        Object item = getItem(position);
        if (item instanceof Comment) {
            return ITEM_TYPE_COMMENT;
        }
        if (item instanceof GameDetail) {
            return ITEM_TYPE_GAME_DETAIL;
        }
        return super.getItemViewType(position);
    }

    protected Object getItem(int position) {
        return mDataSet.get(position);
    }

    @NonNull
    @Override
    public BaseViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        LayoutInflater inflater = LayoutInflater.from(parent.getContext());
        if (viewType == ITEM_TYPE_COMMENT) {
            View itemView = inflater.inflate(R.layout.item_comment, parent, false);
            return new CommentViewHolder(itemView);
        }
        if (viewType == ITEM_TYPE_GAME_DETAIL) {
            View itemView = inflater.inflate(R.layout.item_game_detail, parent, false);
            return new GameDetailViewHolder(itemView);
        }
        return null;
    }

    @Override
    public int getItemCount() {
        return mDataSet.size();
    }
}
```

在游戏详情页面为 `RecyclerView` 创建一个 `GameCommentsAdapter ` 对象。但该方式会让 `GameCommentsAdapter` 变得臃肿，也不满足OCP开闭原则。

### 继承方式实现

扩展一个 `Adapter` 至少要实现 `getItemViewType()`、`onCreateViewHolder()` 等方法，为了复用 `GameComentsAdapter` 类中对评论项，详情页面的 `GameDetailAdapter` 继承该类。

```java
class GameDetailAdapter extends GameCommentsAdapter {
    private static final int ITEM_TYPE_GAME_DETAIL = 2;

    @Override
    public int getItemViewType(int position) {
        Object item = getItem(position);
        if (item instanceof GameDetail) {
            return ITEM_TYPE_GAME_DETAIL;
        }
        return super.getItemViewType(position);
    }

    @NonNull
    @Override
    public BaseViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        if (viewType == ITEM_TYPE_GAME_DETAIL) {
            LayoutInflater inflater = LayoutInflater.from(parent.getContext());
            View itemView = inflater.inflate(R.layout.item_game_detail, parent, false);
            return new GameDetailViewHolder(itemView);
        }
        return super.onCreateViewHolder(parent, viewType);
    }
}
```

### 突然来了一个新需求

产品希望在详情页面添加推荐项，复用首页列表项，如下图所示：

![首页](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/11/15/1671506a98deb975~tplv-t2oaga2asx-image.image)

实现效果如下图所示：

![详情页面增加](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/11/15/1671506a99103e93~tplv-t2oaga2asx-image.image)

Java 是单继承的，`GameDetailAdapter` 已经继承了 `GameComentsAdapter` 类了，无法再继承 `HomeAdapter`。

难道继续在 `GameComentsAdapter` 类中增加 `if` 判断？

### 组合方式

*为了方便阅读，部分代码已省略。*

定义一个模块化的适配器 `Adapter` 类，为了能够被以上 `Adapter` 类所管理，数据项和视图项需要做一些配合：前者继承 `AbstractItem` 类，后者需要继承 `ItemViewHolder` 类。

```java
class Comment extends AbstractItem {}
class GameDetail extends AbstractItem {}
class Game extends AbstractItem {}

class CommentViewHolder extends ItemViewHolder<Comment> {}
class GameDetailViewHolder extends ItemViewHolder<GameDetail> {}
class GameViewHolder extends ItemViewHolder<Game> {}
```

`AbstractItem` 类定义了一个 `type` 属性，代表数据项的类型，会与通过注册的数据项配置信息进行比对，当 `type` 属性值一样时，就会为该数据项 `AbstractItem` 创建对应的视图项 `ViewHolder`。

如果因为 Java 单继承的关系无法继承 `AbstractItem` 类，可以选择实现 `Item` 接口，实现以下方法。

```java
public interface Item {

    void setType(int type);

    int getType();
}
```

此时，数据项和视图项的准备工作已完成，接下来可以组合它们实现需求。

在评论列表页面，创建一个 `Adapter` 实例，并添加评论项功能。

```java
List<Item> dataSet = new ArrayList<>();
dataSet.add(new Comment());
dataSet.add(new GameDetail());

Adapter adapter = new Adapter();
adapter.getManager()
        .register(ITEM_TYPE_COMMENT, CommentViewHolder.class)
        .register(ITEM_TYPE_GAME_DETAIL, GameDetailViewHolder.class)
        .setList(dataSet);
```

在游戏详情页面，创建一个 `Adapter` 实例，并添加游戏项功能。

```java
List<Object> dataSet = new ArrayList<>();
dataSet.add(new Comment());
dataSet.add(new GameDetail());
dataSet.add(new Game());

Adapter adapter = new Adapter();
adapter.getManager()
        .register(ITEM_TYPE_COMMENT, CommentViewHolder.class)
        .register(ITEM_TYPE_GAME_DETAIL, GameDetailViewHolder.class)
        .register(ITEM_TYPE_GAME, GameViewHolder.class)
        .setList(dataSet);
```

当某个页面不再支持评论项时，我们只要删除以下代码即可，不会修改到其他地方，满足OCP设计原则。

```java
dataSet.add(new Comment());
adapter.getManager().unregister(ITEM_TYPE_COMMENT);
```

#### 实现原理

引入 `ItemManager` 接口，统一管理项数据、注册和注销视图项配置信息。

```java
public interface ItemManager {

    ItemManager setList(List<? extends Item> list);

    <T extends ViewHolder> ItemManager register(int type, Class<T> holderClass);

    <T extends ViewHolder> ItemManager register(int type, @LayoutRes int layoutId, Class<T> holderClass);
    
    ItemManager register(ItemConfig config);

    ItemManager unregister(int type);
    
    <T extends Item> T getItem(int position);
}
```

该接口的实现类是 `AdapterDelegate`，主要实现了`getItemViewType`，`onCreateViewHolder`, `onBindViewHolder` 三个 `if-else` 重灾区方法。

```java
public final class AdapterDelegate implements ItemManager {

    public int getItemViewType(int position) {
        Item item = getItem(position);
        ItemConfig adapter = null;
        if (item != null) {
            adapter = registry.get(item.getType());
        }
        if (adapter == null) {
            // TODO
            return 0;
        }
        return adapter.getType();
    }

    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        ItemConfig adapter = registry.get(viewType);
        if (adapter == null) {
            return null;
        }

        int layoutId = adapter.getLayoutId();
        layoutId = layoutId == 0 ? adapter.getType() : layoutId;
        if (layoutId > 0) {
            View itemView = LayoutInflater.from(parent.getContext())
                    .inflate(layoutId, parent, false);
            return createViewHolder(itemView, adapter.getHolderClass());
        }

        return null;
    }

    @SuppressWarnings("unchecked")
    public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
        Item item = getItem(position);
        if (holder instanceof ItemViewHolder) {
            ItemViewHolder viewHolder = (ItemViewHolder) holder;
            viewHolder.setItem(item);
            viewHolder.onViewBound(item);
        }
    }
}
```

使用 `AdapterDelegate` 实现唯一的 `Adapter`，将主要的代码委托给前者。

```java
public class Adapter extends RecyclerView.Adapter<ViewHolder> {

    private AdapterDelegate delegate = new AdapterDelegate();

    @Override
    public int getItemViewType(int position) {
        return delegate.getItemViewType(position);
    }

    @NonNull
    @Override
    public final ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        return delegate.onCreateViewHolder(parent, viewType);
    }

    @Override
    public final void onBindViewHolder(@NonNull ViewHolder holder, int position) {
        delegate.onBindViewHolder(holder, position);
    }
    
    public ItemManager getManager() {
        return delegate;
    }
}
```

## 延伸

在 Java 生态圈之外，有不少组合优于继承的实践。

### Kotlin

Kotlin 语言有 delegation 机制，可以方便开发者使用组合。

 ```kotlin
interface Base {
    fun print()
}

class BaseImpl(val x: Int) : Base {
    override fun print() { print(x) }
}

class Derived(b: Base) : Base by b

fun main(args: Array<String>) {
    val b = BaseImpl(10)
    Derived(b).print()
}
```

#### Kotlin 版 InstrumentedHashSet

```kotlin
class InstrumentedHashSet<E>(val set: MutableSet<E>)
    : MutableSet<E> by set {

    private var addCount : Int = 0

    override fun add(element: E): Boolean {
        addCount++
        return set.add(element)
    }

    override fun addAll(elements: Collection<E>): Boolean {
        addCount += elements.size
        return set.addAll(elements)
    }
}
```

### Go

Go 语言没有继承机制，通过原生支持组合来实现代码的复用。以下分别是 `Reader` 和 `Writer` 接口定义。

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}
```

通过组合可以定义出具备读取和写入的新类型。

```go
type ReadWriter interface {
	Reader
	Writer
}
```

上述的例子是接口组合，也可以是实现组合。*(下面的例子来自 Go in Action 一书)*

```go
type user struct {
	name  string
	email string
}

// notify implements a method that can be called via
// a value of type user.
func (u *user) notify() {
	fmt.Printf("Sending user email to %s<%s>\n",
		u.name,
		u.email)
}

// admin represents an admin user with privileges.
type admin struct {
	user  // Embedded Type
	level string
}

// main is the entry point for the application.
func main() {
	// Create an admin user.
	ad := admin{
		user: user{
			name:  "john smith",
			email: "john@yahoo.com",
		},
		level: "super",
	}

	// We can access the inner type's method directly.
	ad.user.notify()

	// The inner type's method is promoted.
	ad.notify()
}
```

## 推荐书籍

- [Effective Java 中文版第2版](https://book.douban.com/subject/3360807/)
- [重构](https://book.douban.com/subject/4262627/)
- [重构与模式](https://book.douban.com/subject/20393327/)
- [软件设计重构](https://book.douban.com/subject/26854236/)

## 参考资料

- [Effective Java 中文版第2版](https://book.douban.com/subject/3360807/)
- [AdapterDelegates](https://github.com/sockeqwe/AdapterDelegates)
-  [OneAdapter](https://github.com/rome753/OneAdapter)
