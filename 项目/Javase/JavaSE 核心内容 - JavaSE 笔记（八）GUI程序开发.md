![image-20221026154454258](https://s2.loli.net/2022/10/26/Mu6qwoA9fc7THbQ.png)

# GUI程序开发

注意：开始学习之前请确保你完成了《Java SE》篇视频教程。

前面我们已经完成了JavaSE部分的全部内容学习，只不过我们在初学阶段一直都是开发的控制台程序，也就是最原始的命令窗口形式的程序，而Java也可以开发桌面图形化程序，所以我们接着来学习一下Java的图形化界面的开发。

## AWT组件介绍

在Java正式推出的时候，它还包含一个用于基本GUI程序设计的类库，名字叫 Abstract Window Toolkit，简称AWT，抽象窗口工具包，我们可以直接使用Java为我们提供的工具包来进行桌面应用程序的开发。只不过这套工具包依附于操作系统提供的UI，具体样式会根据不同操作系统提供的界面元素进行展示。

实际上我们现代操作系统都是图形化界面，应用程序都是以一个窗口的形式展示出来的，我们可以直接使用鼠标点击窗口内的元素来使用应用程序，相比传统的命令行形式，可方便太多了，比如在Windows和MacOS这两种操作系统下：

![image-20221026164200924](https://s2.loli.net/2022/10/26/fMQDFCmyqIvJeTl.png)

可以看到，不同的操作系统的窗口样式稍微有一些不一样，但是大致的使用方式是差不多的，我们接着来看一下如何使用Java编写简单的桌面图形化程序。

### 基本框架

既然我们要编写一个桌面程序，那么肯定是需要窗口来展示我们程序的内容的，所以说，我们可以使用AWT为我们提供的组件来创建窗口：

```java
public static void main(String[] args) {
    Frame frame = new Frame();   //Frame是窗体，我们只需要创建这样一个对象就可以了，这样就会直接创建一个新的窗口
    frame.setSize(500, 300);   //可以使用setSize方法设定窗体大小
    frame.setVisible(true);    //默认情况下窗体是不可见的，我们如果要展示出来，还需要设置窗体可见性
}
```

可以看到，桌面的左上角已经展示出我们的窗口了：

![image-20221026165600076](https://s2.loli.net/2022/10/26/ZSQs1NhWlJeyMmi.png)

在不同的操作系统下，窗口的样式会不同。

我们可以通过Frame的各种方法来设置窗口的各项属性：

```java
public static void main(String[] args) {
    Frame frame = new Frame();
    frame.setTitle("我是标题");   //设置窗口标题
    frame.setSize(500, 300);    //设置窗口大小
    frame.setBackground(Color.BLACK);   //设置窗口背景颜色
  	frame.setResizable(false);    //设置窗口大小是否固定
  	frame.setAlwaysOnTop(true);    //设置窗口是否始终展示在最前面
    frame.setVisible(true);    //注意，只有将可见性变为true时才会展示出这个窗口，否则窗口是隐藏的
}
```

实际上当我们创建一个窗口之后，会在其他线程中进行处理，包括窗口的绘制、窗口事件的监听等，所以说我们的主线程不会卡住。

实际上我们的程序打开都是默认居中显示的，所以说我们可以调整一下窗口的位置：

```java
frame.setLocation(100, 200);   //setLocation可以调整窗口位置
```

注意，这里的窗口位置以及窗口大小都是以像素为单位。整个屏幕有多少个像素，是根据各位小伙伴电脑的显示器屏幕分辨率来决定的，比如我们的电脑显示器屏幕分辨率为 1920 x 1080，那么我们显示器就可以显示长为1920个像素，宽1080个像素的矩形，只要是在这个范围内的窗口，都可以显示到屏幕上：

![image-20221026170449235](https://s2.loli.net/2022/10/26/CknumyFjpz659Ya.png)

那么问题就来了，如果现在我们希望将这个窗口居中，就需要手动调整位置，但我们是要去适配各种分辨率的显示器才可以，不然到其他分辨率下，就无法居中了，我们可以动态获取分辨率来进行位置计算：

```java
public static void main(String[] args) {
    Frame frame = new Frame("我是标题");
    frame.setSize(500, 300);

    Dimension screenSize = Toolkit.getDefaultToolkit().getScreenSize();  //获取到屏幕尺寸
    int x = (int) ((screenSize.getWidth() - frame.getWidth()) / 2);   //居中位置就是：屏幕尺寸/2 - 窗口尺寸/2
    int y = (int) ((screenSize.getHeight() - frame.getHeight()) / 2);

    frame.setLocation(x, y);   //位置设置好了之后再展示出来
    frame.setVisible(true);
}
```

这样我们的窗口打开之后默认就是居中放置的了，是不是感觉用Java开发图形界面好像也不是那么难？

得益于Java已经为我们封装好了各种方法，所以说要实现什么功能直接调用对应的方法即可，比如我们想要个性化光标，我们可以使用`setCursor`方法来实现，JDK已经为我们提供了一些预设的光标样式：

![image-20221027151713661](https://s2.loli.net/2022/10/27/drC1nx2NSK9Ewaf.png)

设定光标样式后，当我们的鼠标移动到这个窗口内部时，就会变成我们设定好的光标样式了。

有关其他方法，这里暂时不进行介绍。

### 监听器

我们可以为窗口添加一系列的监听器，监听器会监听窗口中发生的一些事件，比如我们点击关闭窗口、移动鼠标、鼠标点击等，当发生对应的事件时，就会通知到对应的监听器进行处理，从而我们能够在发生对应事件时进行对应处理。

![image-20221027161611050](https://s2.loli.net/2022/10/27/DAz1hnUekV6RNqd.png)

比如我们现在希望点击关闭按钮关闭当前的窗口，但是我们发现默认情况下实际上是关不掉的，因为我们并没有对关闭事件进行处理，默认情况下对于这种点击时没有设定任何动作的，万一我们点了之后并不是要关闭窗口呢。要实现关闭窗口，我们可以使用`addXXXListener`来添加对应的事件监听器，比如窗口相关的操作那么就是WindowListener：

![image-20221027155830335](https://s2.loli.net/2022/10/27/IiwomHF7YWe8Vuh.png)

这里我们可以给一个接口实现，或是使用对应的适配器（适配器模式是设计模式中的一种写法，因为接口中要实现的方法太多，但是实际上我们并不需要实现那么多，只需要实现对应的即可，所以说就可以使用适配器）我们只需要重写对应的方法，当发生对应事件时就会自动调用我们已经实现好的方法：

```java
frame.addWindowListener(new WindowAdapter() {
    @Override
    public void windowClosing(WindowEvent e) {   //windowClosing方法对应的就是窗口关闭事件
        frame.dispose();    //当我们点击X号关闭窗口时，就会自动执行此方法了
        //使用dispose方法来关闭当前窗口
    }

    @Override
    public void windowClosed(WindowEvent e) {   //对应窗口已关闭事件
        System.out.println("窗口已关闭！");   //当窗口成功关闭后，会执行这里重写的内容
      	System.exit(0);    //窗口关闭后退出当前Java程序
    }
});
```

我们可以来看看效果，现在我们点击X号关闭窗口就可以成功执行了，并且窗口关闭后我们的Java程序就结束了。当然，监听器可以添加多个，并不是只能有一个。

这里总结一下窗口常用的事件：

```java
public interface WindowListener extends EventListener {
    public void windowOpened(WindowEvent e);   //当窗口的可见性首次变成true时会被调用
    public void windowClosing(WindowEvent e);   //当以后企图关闭窗口（也就是点击X号）时被调用
    public void windowClosed(WindowEvent e);   //窗口被我们成功关闭之后被调用
    public void windowIconified(WindowEvent e);    //窗口最小化时被调用
    public void windowDeiconified(WindowEvent e);   //窗口从最小化状态变成普通状态时调用
    public void windowActivated(WindowEvent e);    //当窗口变成活跃状态时被调用
    public void windowDeactivated(WindowEvent e);   //当窗口变成不活跃时被调用
}
```

除了监听窗口相关的动作之外，我们也可以监听鼠标、键盘等操作的事件，比如键盘事件：

```java
frame.addKeyListener(new KeyAdapter() {
    @Override
    public void keyTyped(KeyEvent e) {    //监听键盘输入事件，当我们在窗口中敲击键盘输入时会触发
        System.out.print(e.getKeyChar());   //可以通过KeyEvent对象来获取当前事件输入的对应字符
    }
});
```

键盘事件甚至可以细致到键盘按键的几种状态：

```java
public interface KeyListener extends EventListener {
    public void keyTyped(KeyEvent e);   //当一个按键按下之后触发（感觉跟下面这个没啥区别）
    public void keyPressed(KeyEvent e);   //当一个按键按下后触发（按下之后如果不松开会连续触发此事件）
    public void keyReleased(KeyEvent e);   //当一个按键按下然后松开后触发
}
```

我们也可以监听鼠标相关的事件，比如当鼠标点击我们界面上某一个位置时，我们就可以获取一下：

```java
frame.addMouseListener(new MouseAdapter() {
    @Override
    public void mouseClicked(MouseEvent e) {   //mouseClicked是监听鼠标点击事件（必须要用真的鼠标点击，不知道为啥，笔记本的触摸板不行，可能是MacOS的BUG吧）
        System.out.println("鼠标点击："+e.getX()+","+e.getY());
    }
});
```

这样，当我们点击窗口中的某个位置时，就可以获取对应的坐标并打印出来：

![image-20221027164500070](https://s2.loli.net/2022/10/27/VQm7FjSNidLhI1r.png)

注意这里的坐标并不是按照我们在数学中学习的平面直角坐标系来的，它的X轴是从左往右，但是Y轴是从上往下，原点也不是整个屏幕开始，而是我们的窗口左上角。所以说当我们点击右下角时，就会得到一个接近于窗口大小的坐标了。

我们也可以获取鼠标是使用哪个键点击的，我们的鼠标一般情况下有三个按键：

* BUTTON1   -   鼠标左键，也是我们用的最多的键
* BUTTON2   -   鼠标中键，一般是鼠标滚轮，也是是可以点击的（不会有人以为鼠标滚轮只能滚不能按吧）
* BUTTON3   -   鼠标右键，右键一般就是辅助点按，展开各种选项等

如果是游戏鼠标，也许能监听到一些其他的按键，这里我们就不测试了，我们来尝试监听一下：

```java
frame.addMouseListener(new MouseAdapter() {
    @Override
    public void mouseClicked(MouseEvent e) {
        System.out.println("鼠标点击："+e.getButton());
    }
});
```

鼠标滚动事件也可以进行监听：

```java
frame.addMouseWheelListener(new MouseAdapter() {
    @Override
    public void mouseWheelMoved(MouseWheelEvent e) {
        System.out.println(e.getScrollAmount());    //获取滚动数量
    }
});
```

MacOS下的鼠标滚动是平滑滚动，会触发很多次，不像Windows下是一格一格的僵硬滚动。

通过使用这些监听器，我们就可以更好的控制我们的GUI程序了。

### 常用组件

前面我们介绍了监听器，我们接着来看看常用的一些组件，那么什么是组件呢？

组件实际上是AWT为我们预设好的一些可以直接使用的界面元素，比如按钮、文本框、标签等等，我们可以使用这些已经帮我们写好的组件来快速拼凑出一个好看且功能强大的程序：

![image-20221027170224462](https://s2.loli.net/2022/10/27/D6hslN2pHybmVdF.png)

在开始学习组件之前，我们先将布局设定为`null`（因为默认情况下会采用BorderLayout作为布局）有关布局我们会在下一部分中进行介绍，这节课我们先介绍没有布局的情况下如何使用这些组件。

```java
frame.setLayout(null);
```

首先我们来介绍一下最简单的组件，标签组件相当于一个普通的文本内容，我们可以将自己的标签添加到窗口中：

```java
Label label = new Label("我是标签");   //添加标签只需要创建一个Label对象即可
label.setLocation(20, 50);   //注意，必须设定标签的位置和大小，否则无法展示出来
label.setSize(100, 20);
frame.add(label);    //使用add方法添加组件到窗口中
```

注意，组件的位置是以整个窗口的左上角为原点开始的（整个窗口指的是包括标题栏在内）所以说我们如果想要设置组件的位置，我们还得注意加上标题栏的高度，否则会被标题栏遮挡：

![image-20221027175842110](https://s2.loli.net/2022/10/27/VjCdNbAUIi5R61Z.png)

我们可以自由修改文本的字体和大小：

```java
//直接构造并传入一个Font对象即可
label.setFont(new Font("SimSong", Font.BOLD, 15));   //Font构造方法需要字体名称、字体样式（加粗、斜体）、字体大小
```

注意必须是操作系统已经安装的字体才支持展示，如果各位小伙伴不知道操作系统有哪些字体，可以使用：

```java
GraphicsEnvironment.getLocalGraphicsEnvironment().getAllFonts()
```

来获取所有的系统字体：

![image-20221027181909908](https://s2.loli.net/2022/10/27/Fsj8PqHryUYdgeC.png)

这里我们直接使用前面的`family`即可，比如我们要使用宋体，那么就输入其名称：

```java
label.setFont(new Font("Songti SC", Font.BOLD, 15));
```

![image-20221027182010485](https://s2.loli.net/2022/10/27/zy4L2T67rGQe3HI.png)

可以看到字体已经成功修改了，当然，为了方便，如果我们的窗口中有很多的标签都想统一使用某一个字体，我们可以直接对窗口设定字体，那么只要是添加到窗口中的组件都会默认使用这个字体，除非单独指定组件字体。

要修改字体的颜色也很简单，我们可以使用：

```java
label.setBackground(Color.BLACK);    //setBackground依然是背景颜色，注意背景填充就是我们之前设定的大小
label.setForeground(Color.WHITE);    //setForeground是设定字体颜色
```

![image-20221027183745934](https://s2.loli.net/2022/10/27/R5cDrYxoKtpCJk6.png)

我们接着来认识一下下一个组件，这个组件的名字叫做按钮，实际上按钮也是我们经常会使用的一个组件：

```java
Button button = new Button("点击充值");   //Button是按钮组件
button.setBounds(20, 50, 100, 50);
frame.add(button);
```

这样就可以添加一个按钮到我们的窗口中了：

![image-20221027182903783](https://s2.loli.net/2022/10/27/gArLdNTI1ClWh5K.png)

只不过，既然是按钮，那么肯定要添加一些点击动作才可以，比如点击按钮之后打印充值成功：

```java
button.addActionListener(e -> System.out.println("充值成功"));  //addActionListener就是按钮点击监听器
```

是不是感觉还是很简单？当然，如果要修改按钮的字体或是颜色，依然使用之前的方式即可。

只不过光有按钮似乎太单调了一点，我们接着来认识下一个组件：

```java
TextField field = new TextField();   //TextField是文本框
field.setBounds(20, 50, 100, 25);
frame.add(field);
```

![image-20221027184138604](https://s2.loli.net/2022/10/27/ZhpojvR6u5DTmVP.png)

 我们经常要在一些软件上登录，那么就要输入我们的用户名和密码，所以说文本框的作用还是非常明显的，我们也可以通过AWT组件来实现这些功能，我们可以来试试看：

```java
TextField field = new TextField();
field.setBounds(20, 50, 200, 25);
frame.add(field);

Button button = new Button("点击登录");
button.setBounds(20, 80, 100, 50);
//点击按钮直接获取文本框中的文本内容，只需要调用getText方法即可
button.addActionListener(e -> System.out.println("输入的用户名是："+field.getText()));
frame.add(button);
```

我们来试试看吧：

![image-20221027184618359](https://s2.loli.net/2022/10/27/7re3aBis2jW9wvP.png)

![image-20221027184627653](https://s2.loli.net/2022/10/27/Erb6npLZqsCPGzQ.png)

是不是感觉有内味了？当然，可能会有小伙伴觉得如果我们输入密码的话，不应该将展示的文字隐藏起来吗？我们可以这样：

```java
field.setEchoChar('*');   //setEchoChar设定展示字符，无论我们输入的是什么，最终展示出来的都是我们指定的字符
```

![image-20221027184814288](https://s2.loli.net/2022/10/27/Er4wqYUNfAy2HBS.png)

当然，我们在获取输入的文本时还是输入的文本本身，不会变成展示的文本，只是一个视觉效果而已。这样，我们就可以将密码框做出来了。各位小伙伴可以尝试做一个登录界面。

但是肯定有小伙伴疑问，不是还有一个记住密码的勾选框吗？安排：

```java
Checkbox checkbox = new Checkbox("记住密码");
checkbox.setBounds(20, 50, 100, 30);   //这个大小并不是勾选框的大小，具体的勾选框大小要根据操作系统决定，跟Label一样，是展示的空间大小
frame.add(checkbox);
```

最终展示出来的效果就是：

![image-20221027185748324](https://s2.loli.net/2022/10/27/pErwuKAGOa3dQ5l.png)

效果还是挺不错的，我们也可以设定一个多选框：

```java
CheckboxGroup group = new CheckboxGroup();   //创建勾选框组

Checkbox c1 = new Checkbox("选我");
c1.setBounds(20, 50, 100, 30);
frame.add(c1);

Checkbox c2 = new Checkbox("你干嘛");
c2.setBounds(20, 80, 100, 30);
frame.add(c2);

c1.setCheckboxGroup(group);    //多个勾选框都可以添加到勾选框组中
c2.setCheckboxGroup(group);
```

![image-20221027190207441](https://s2.loli.net/2022/10/27/y8713x9VBlCaQm6.png)

我们可以使用`getSelectedCheckbox`方法来获取已经被选中的勾选框：

```java
System.out.println(group.getSelectedCheckbox());
```

常用组件就暂时介绍到这里。

### 布局和面板

前面我们介绍了各种各样的组件，现在我们就可以利用这些组件来拼凑一个好看的程序了。

只不过，如果不使用布局，那么我们只能手动设置组件的位置以及大小，这就使得我们的程序在尺寸的设计上很有限，因为一旦窗口的大小发生变化，我们的组件依然是会放置在原本的位置上，要保证我们的设计不被破坏就只能固定窗口大小，但是很多应用都是支持放大和缩小的，并且在不同的大小下组件会自己调整位置：

![image-20221028135701447](https://s2.loli.net/2022/10/28/ro7Cxi5Oe8wuALv.png)

可以看到窗口的大小可以自由移动并且组件的位置会根据窗口大小自己进行调整。

这正是因为使用了布局实现的，布局可以根据自己的一些性质，对容器（这里可以是我们的窗口）内部的组件自动进行调整，包括组件的位置、组件的大小等，Java为我们提供了各种各样的布局管理器，我们来看看吧。

默认情况下，我们的窗口采用的是边界布局（BorderLayout）这种布局方式支持将组件放置到五个区域：

```java
frame.setLayout(new BorderLayout());   //使用边界布局
frame.add(new Button("1号按钮"), BorderLayout.WEST);  //在添加组件时，可以在后面加入约束
frame.add(new Button("2号按钮"), BorderLayout.EAST);
frame.add(new Button("3号按钮"), BorderLayout.SOUTH);
frame.add(new Button("4号按钮"), BorderLayout.NORTH);
frame.add(new Button("5号按钮"), BorderLayout.CENTER);
```

注意，约束只有在当前容器为对应布局时才可以使用。这里我们采用的是边界布局，边界布局可以将组件设定到五个区域：

![image-20221028140816161](https://s2.loli.net/2022/10/28/ZigtAkDbrMVqjWz.png)

可以看到，分别在东、南、西、北、中心位置都可以添加组件，组件的大小会被自动调整，并且随着我们的窗口大小变化，组件的大小也会跟着自动调整，是不是感觉挺方便的？边界布局的性质：

*  BorderLayout布局的容器某个位置的某个组件会直接充满整个区域。
* 如果在某个位置重复添加组件，只有最后一个添加的组件可见。
* 缺少某个位置的组件时，其他位置的组件会延伸到该位置。

我们还可以调整组件之间的间距：

```java
BorderLayout borderLayout = new BorderLayout();
borderLayout.setHgap(50);   //Hgap是横向间距
borderLayout.setVgap(50);   //Vgap是纵向间距
```

调整之后，边距就非常明显了：

![image-20221028143042506](https://s2.loli.net/2022/10/28/XQqnd6GHa7hVOtR.png)

我们接着来认识一下下一个布局，FlowLayout 流式布局，流式布局实际上就是按顺序排列的一种布局：

```java
frame.setLayout(new FlowLayout());   //采用流式布局
frame.add(new Button("1号按钮"));
frame.add(new Button("2号按钮"));
frame.add(new Button("3号按钮"));
```

采用流式布局后，按钮会根据内容大小，自动调整为对应的大小，并且他们之间是有间距的：

![image-20221028142144585](https://s2.loli.net/2022/10/28/471ED3GaefjzHy8.png)

当我们对窗口大小进行调整时，流式布局也会进行自动调整：

![image-20221028142326191](https://s2.loli.net/2022/10/28/hJrBtcVj7MDGqfw.png)

我们也可以在设定流式布局时指定对齐模式：

```java
frame.setLayout(new FlowLayout(FlowLayout.RIGHT));   //指定为右对齐
```

对齐方式会直接决定组件的排列方式：

![image-20221028142506961](https://s2.loli.net/2022/10/28/QlU8IPoVA2j4E6t.png)

我们同样可以使用Hgap和Vgap来调整组件之间的间距：

```java
FlowLayout flowLayout = new FlowLayout();
flowLayout.setHgap(50);
flowLayout.setVgap(0);
```

![image-20221028143230216](https://s2.loli.net/2022/10/28/vVN4O8XynAdW5Jb.png)

我们接着来看卡片布局，CardLayout对象将卡片作为一个容器中的每个组件，这个卡片布局怎么说呢，有点像iOS14新出的叠放小组件（安卓应该也有）就像很多张卡片叠在一起，每次只能看到最顶上的这张卡片，但是我们可以将下层的卡片切到顶上来：

![image-20221028143949323](https://s2.loli.net/2022/10/28/CqE9FkVSMJXOLY8.png)

卡片布局就是这样，我们可以添加多个组件：

```java
CardLayout layout = new CardLayout();
frame.setLayout(layout);
frame.add(new Label("我是1号"));
frame.add(new Label("我是2号"));

frame.setVisible(true);

while (true) {
    Thread.sleep(3000);
    layout.next(frame);    //我们需要使用CardLayout对象来进行切换
}
```

这里我们每三秒钟切换一次卡片，可以看到我们添加的标签每三秒就会变化一次，实际上我们可以利用卡片布局来做一个类似跑马灯的效果，是不是感觉很神奇？

我们接着来看网格布局，GridLayout以矩形网格的形式对组件进行管理：

```java
frame.setLayout(new GridLayout());
for (int i = 0; i < 10; i++)
    frame.add(new Button(i + "号按钮"));
```

这种布局就很好理解了，默认情况下会生成一行按格子划分的相等区域：

![image-20221028145118733](https://s2.loli.net/2022/10/28/joR1s467rFzJqOE.png)

我们也可以手动指定行数和列数：

```java
GridLayout gridLayout = new GridLayout();
gridLayout.setRows(2);
frame.setLayout(gridLayout);
for (int i = 0; i < 10; i++)
    frame.add(new Button(i + "号按钮"));
```

![image-20221028145330522](https://s2.loli.net/2022/10/28/DtnRaCJh7qzc4w3.png)

所有的组件都会整齐排列在网格中。

最后一种布局是GridBagLayout，是最灵活的布局管理器，它同样是按照网格进行划分，但是一个组件可以同时占据多个网格。这种情况其实也是经常会出现的，比如计算器上的按钮虽然看起来也是按照网格排列的，但是有些按钮同时占据了横向或是纵向的两个网格，这种情况使用GridBagLayout布局就可以很好的处理：

![image-20221028145752545](https://s2.loli.net/2022/10/28/JNmjEVWIG2nkxbf.png)

虽然这个布局很强大，但是用起来也是很麻烦的，所以说这里就不做讲解了，感兴趣的小伙伴可以自行了解。

虽然认识了这么多的布局，但是我们发现，很多应用程序并不只是由单一的布局组成的，而是多种布局相互嵌套的结果，比如我们的IDEA界面，就不仅仅是一个布局完成的（这里只是举个例子）而是多种布局在嵌套使用：

![image-20221028151242522](https://s2.loli.net/2022/10/28/9ZfvOxThctdPS6N.png)

但是只有我们的窗口才能设置布局啊，总不可能让多个窗口拼接在一起吧？实际上除了窗口可以作为容器之外，我们也可以使用其他的容器，这时，我们就需要用到面板。

类面板是最简单的容器类，它跟窗口一样，可以提供一个空间，同样可以随意添加组件到面板中，只不过面板本身也是一个组件，所以说面板是可以放到其他容器中的容器，就像：

![image-20221028151701189](https://s2.loli.net/2022/10/28/jVGTNmd3i2ZRg5h.png)

![image-20221028151845514](https://s2.loli.net/2022/10/28/6PGem8qMrV7NOZS.png)

面板本身也是容器，所以说也可以单独设置面板内部的布局，比如现在我们想要分两个区域，上半部分区域是流式布局，下半部分区域采用网格布局，那么我们就可以先将窗口采用网格布局，并在上下各添加一个面板：

```java
GridLayout layout = new GridLayout();   //先设置整个窗口的布局
layout.setRows(2);     //设置行数为2，一会就会分成两行了
frame.setLayout(layout);

Panel top = new Panel();     //接着我们创建一下上半部分的面板和下半部分的面板
top.setBackground(Color.PINK);   //添加一个背景颜色方便区分
frame.add(top);

Panel bottom = new Panel();
bottom.setBackground(Color.ORANGE);
frame.add(bottom);
```

这样，我们的两个面板就按照网格布局，被分成了上下两部分：

![image-20221028152352861](https://s2.loli.net/2022/10/28/agMjZkqrSGUm9Ld.png)

接着我们就可以分别在上半部分的面板和下半部分的面板中进行单独配置了：

```java
Panel top = new Panel();
top.setBackground(Color.PINK);
top.setLayout(new FlowLayout());    //面板默认会采用FlowLayout，所以说这里指不指定都一样
for (int i = 0; i < 5; i++)    //面板就像窗口一样，可以设定布局和添加组件
    top.add(new Button("流式"+i));
frame.add(top);

Panel bottom = new Panel();
bottom.setBackground(Color.ORANGE);
bottom.setLayout(new GridLayout());   //下半部分我们采用网格布局
for (int i = 0; i < 5; i++)
    bottom.add(new Button("网格"+i));
frame.add(bottom);
```

这里我们将上半部分面板设定为流式布局，下半部分面板设定为网格布局：

![image-20221028152617119](https://s2.loli.net/2022/10/28/RbiFpTDCEaN5fPl.png)

利用面板，我们就可以实现各种布局的自由组合，当然，面板在后面还会有更多的用处。

### 滚动面板和列表

有些时候，我们的窗口大小可能并不能完全显示内部的内容，比如出现了一张很大的图片。

![image-20221028153201386](https://s2.loli.net/2022/10/28/7mKakMLhz95VbIp.png)

此时就会出现滚动条来让我们进行拖拽，这样就可以向下滑动查看没有完全展示出来的内容了。而我们之前开发的程序都没办法做到这样的滚动，超出部分会直接无法显示。

AWT也为我们提供了滚动面板组件，滚动面板也是一个容器，但是我们无法修改它的布局，它只能容纳单个组件，比如展示一个图片、或者是列表等，我们也可以将其与Panel配合使用，比如：

```java
ScrollPane scrollPane = new ScrollPane();   //创建滚动面板
frame.add(scrollPane);

GridLayout layout = new GridLayout();    //创建滚动面板内部的要展示的面板
layout.setRows(20);
Panel panel = new Panel();
panel.setLayout(layout);
for (int i = 0; i < 20; i++)
    panel.add(new Button("我是按钮"+i));   //为面板添加按钮
scrollPane.add(panel);
```

可以看到，无法显示的部分会自动变成滚动面板，我们滑动就可以展示了：

![image-20221028155050727](https://s2.loli.net/2022/10/28/ZDa92CJVf7TbGk4.png)

这里需要特别提一下，我们看到这里的按钮大小采用的是自动生成的大小，但是如果我们希望按钮的大小按照我们喜欢的来怎么办呢？我们知道，使用布局之后，组件的大小实际上是自动决定的，只有未使用布局的情况下才能自由更改组件大小，那么我们怎么才能干预呢？我们可以为组件设定一个建议的大小：

```java
for (int i = 0; i < 20; i++) {
    Button button = new Button("我是按钮" + i);
    button.setPreferredSize(new Dimension(100, 50));   //设置首选大小
    panel.add(button);
}
```

当布局管理器在自动调整内部组件大小时，如果不是必须要按照布局大小来展示或者是高度或宽度不确定，那么就会采用我们建议的大小展示，比如这里只能确定宽度，而高度是不确定的，那么就可以使用我们建议的大小来展示：

![image-20221028155443331](https://s2.loli.net/2022/10/28/SQZapDy6vdLkHNx.png)

当然，首选大小可能不太好理解，还需要各位小伙伴多多尝试才能理解。

实际上滚动面板的最佳搭档就是List列表（注意这里的列表不是我们集合类里面学习的列表，而是展示出来的列表组件）

```java
List list = new List();   //注意是awt包下的List，别导错了
list.add("小糍粑");
list.add("锅巴洋芋");
list.add("手抓饼");
list.add("凉面");
list.setMultipleMode(true);   //是否开启多选模式
```

列表组件就像是一个选择列表一样：

![image-20221028160340931](https://s2.loli.net/2022/10/28/ieDtpQqdkBzhsKF.png)

列表会将元素依次展示出来，我们可以选择列表中的某一项：

```java
list.addItemListener(System.out::println);
```

列表可以添加监听器，当我们选择某个物品时，就会自动触发：

![image-20221028160611113](https://s2.loli.net/2022/10/28/LXIvYhnFVBlQTGt.png)

列表就很时候拿来做文件列表。

### 菜单栏

前面我们认识了各种各样的组件，我们接着来看菜单，实际上各位小伙伴会发现我们的程序上方一般都会有一排菜单：

![image-20221028161209224](https://s2.loli.net/2022/10/28/Konar26QHWMTwqd.png)

在MacOS下是整合到状态栏中的：

![image-20221028161239672](https://s2.loli.net/2022/10/28/G3NiRaBMkJLneVl.png)

这些菜单一般都会包含我们程序中的一些基本操作，实际上我们的程序中很多地方都会使用到下拉菜单：

![image-20221028161118028](https://s2.loli.net/2022/10/28/hc354p1ri6NmGgA.png)

而我们编写AWT程序也可以添加这样的菜单，只需要为窗口设定一个菜单栏即可：

```java
MenuBar bar = new MenuBar();    //创建菜单栏 
Menu menu = new Menu("我是1号菜单");
menu.add(new MenuItem("测试1"));
menu.add(new MenuItem("测试2"));
bar.add(menu);
frame.setMenuBar(bar);    //为窗口设定刚刚定义好的菜单栏
```

设定好MenuBar之后，我们的程序就有菜单了：

![image-20221028161741397](https://s2.loli.net/2022/10/28/wIWdRo2velTj1VS.png)

虽然有点丑，但是确实是内味，不过还是MacOS下好看：

![image-20221028161910928](https://s2.loli.net/2022/10/28/7Dq6L1hbreYIy39.png)

我们着重来看一下MenuItem，这是我们菜单的每一个选项，我们可以为其添加监听器来监听用户是否点击：

```java
MenuItem item = new MenuItem("测试1");
item.addActionListener(e -> System.out.println("一号选项被点击了！"));
menu.add(item);
```

其实跟我们之前学习的按钮是差不多的：

![image-20221028162602244](https://s2.loli.net/2022/10/28/KskDE9J2QdRtlvV.png)

我们还可以为菜单中的选项设定快捷键：

```java
MenuItem item = new MenuItem("测试1");
item.setShortcut(new MenuShortcut('A'));   //MenuShortcut就是指定快捷键组合，默认情况下是Ctrl+指定按键
//item.setShortcut(new MenuShortcut('A', true));   //第二个参数指定为true表示需要Ctrl+Shift+指定按键
```

这里的效果就是Ctrl+A触发快捷键：

![image-20221030173320786](https://s2.loli.net/2022/10/30/lwrjgRxu46UXHZk.png)

当然，除了这种普通的菜单选项之外，还有可以勾选的：

```java
menu.add(new CheckboxMenuItem("测试2"));
```

CheckboxMenuItem是可以勾选的选项，它能够对状态进行记录，我们点击选项之后会变成勾选状态：

![image-20221028162655033](https://s2.loli.net/2022/10/28/Q1RgUn7ejZXzH5E.png)

实际上要添加这样的菜单栏还是挺简单的的，我们接着来看弹出菜单，弹出菜单其实也经常出现，比如我们要新建一个类，我们就可以右键对应的包：

![image-20221028214019648](https://s2.loli.net/2022/10/28/IKe1NL5wm834WdP.png)

弹出一个浮在窗口之上的，并且可以进行选择的菜单，这个就是弹出菜单。

比如我们想要实现右键窗口任意位置都弹出菜单：

```java
PopupMenu menu = new PopupMenu();    //创建弹出菜单
menu.add(new MenuItem("选项1"));   //每一个选项依然是使用MenuItem
menu.add(new MenuItem("选项2"));
frame.add(menu);    //注意，弹出菜单也要作为组件加入到窗口中（但是默认情况下不显示）

frame.addMouseListener(new MouseAdapter() {
    @Override
    public void mousePressed(MouseEvent e) { 
        if (e.getButton() == MouseEvent.BUTTON3) {  //监听鼠标右键
            menu.show(frame, e.getX(), e.getY());   //要展示弹出菜单，我们只需要调用show方法即可
          	//注意，第一个参数必须是弹出菜单所加入的窗口或是窗口中的任意一个组件
          	//后面的坐标就是相对于这个窗口或是组件的原点（左上角）这个位置进行弹出
          	//我们这里写的就是相对于当前窗口的左上角，鼠标点击位置的x、y位置弹出窗口
        }
    }
});
```

我们来看看实际效果吧：

![image-20221028215651667](https://s2.loli.net/2022/10/28/tMRbdjE6ZhHuPaQ.png)

这样，我们就可以设计出更加高级的程序了。

### 对话框

有些时候，我们点击关闭按钮之后，窗口并不会直接关闭，而是会弹出一个对话框询问我们是否要退出，比如我们使用记事本编辑完之后未保存就关闭记事本，就会提示我们：

![image-20221028220721666](https://s2.loli.net/2022/10/28/VUshJzZXxC51gpb.png)

实际上像这样弹出的的一个对话框，在很多时候都很关键，我们也可以使用AWT为我们提供的对话框，比如我们现在希望在关闭窗口时询问我们是否真的要关闭：

```java
Dialog dialog = new Dialog(frame, "我是对话框", true);   
//第一个参数是父窗口或是父对话框（没错，对话框也可以由对话框唤起）
//最后一个参数是当对话框展示时，是否让父窗口（对话框）无法点击
dialog.setSize(200, 80);

frame.addWindowListener(new WindowAdapter() {
    @Override
    public void windowClosing(WindowEvent e) {
        dialog.setVisible(true);
    }
});
```

当我们点击关闭时：

![image-20221028223431754](https://s2.loli.net/2022/10/28/VmajcUlSD6GPsrv.png)

可以看到这里确实弹出了一个对话框（这个对话框无法最小化到图标）也就是说我们只能通过操作对话框来关闭它。

只不过就单单是这样的一个对话框太过单调了，我们可以为其添加一些按钮之类的东西：

```java
Dialog dialog = new Dialog(frame, "我是对话框", true);
dialog.setResizable(false);
dialog.add(new Label("确定是否要退出程序？"), BorderLayout.NORTH);   //对话框默认采用的是边界布局
dialog.add(new Button("取消"), BorderLayout.WEST);
dialog.add(new Button("不退出"), BorderLayout.EAST);
dialog.setSize(200, 80);
```

这样我们退出时，就有对应的提示了：

![image-20221028224410637](https://s2.loli.net/2022/10/28/3v7ZJio9mMnK8zk.png)

对话框就像一个特殊的窗口一样，各位小伙伴可以自由发挥。

有些时候我们在使用应用程序的时候，可能需要我们去选择电脑上的一些文件，这个时候我们就可以使用文件对话框：

```java
FileDialog dialog = new FileDialog(frame, "请选择一个文件", FileDialog.LOAD);  //选择文件对话框类型，可以是加载文件或是保存文件

frame.addWindowListener(new WindowAdapter() {
    @Override
    public void windowClosing(WindowEvent e) {
        dialog.setVisible(true);
    }
});
```

文件对话框是根据操作系统提供的文件选择器决定的：

![image-20221028224815769](https://s2.loli.net/2022/10/28/TuHNLsmokZvMhIR.png)

我们可以通过文件对话框选择一个文件：

```java
FileDialog dialog = new FileDialog(frame, "请选择一个文件", FileDialog.LOAD);

frame.addWindowListener(new WindowAdapter() {
    @Override
    public void windowClosing(WindowEvent e) {
        dialog.setVisible(true);   //注意，需要将对话框展示出来之后，才能进行选择
      	//选择完成之后getDirectory和getFile方法就可以返回结果了，否则会阻塞
        System.out.println("选择的文件为："+dialog.getDirectory() + dialog.getFile());
    }
});
```

选择文件之后，我们可以通过对话框直接获取到对应的文件：

![image-20221028225343739](https://s2.loli.net/2022/10/28/uRjWKPgFxCcGUvr.png)

是不是感觉还是挺简单的？

### 自定义组件

除了使用官方提供的这些组件之外，我们也可以自己创建一些组件来使用，比如官方没有提供图片组件，我们可以自己编写一个图片组件用于在窗口中展示我们的图片。

要自己编写一个组件，需要完成下面的步骤：

* 必须继承自Component类，表示这是一个AWT组件。
* 需要自己实现`paintComponent`方法，这个方法就是组件的绘制方法，最终绘制出来的结果就是展示出来的结果了。

首先我们先把最简单的事情做了：

```java
public class ImageView extends Component {   //继承自Component表示是一个组件
    
    public ImageView(){}

    @Override
    public void paint(Graphics g) {    //重写paint方法，这个方法就是组件的绘制方法
        
    }
}
```

我们可以先创建一个这个组件并放到我们的窗口中：

```java
frame.add(new ImageView()); 
```

这里用的是边界布局，默认情况下组件会被添加到中心，占满整个窗口。但是由于我们并没有编写任何绘制内容，所以说组件是空白的一片。

我们来看看这个`paint`方法该如何重写，这个方法实际上是在窗口绘制时自动调用，那么到底什么是绘制呢？实际上绘制就是需要我们进行画图操作，当窗口首次展示或是修改大小时就会调用这个方法绘制组件（使用过OpenGL的小伙伴应该能够很容易上手）

这个方法给了我们一个Graphics对象，实际上这个对象就是我们用于绘制图形的工具，比如我们这个组件需要绘制的是一个矩形：

```java
@Override
public void paint(Graphics g) {   //各位小伙伴可以将Graphics看做一只画笔，我们想让画笔做什么就做什么
    g.setColor(Color.BLACK);      //我们可以先将画笔切换为黑色
    g.fillRect(0, 0, getWidth(), getHeight());   //drawRect就是绘制矩形区域，这里的x和y是相对于当前组件的位置来的
}
```

我们来看看最后会绘制成啥样：

![image-20221028232701565](https://s2.loli.net/2022/10/28/qQweM6DprRJjCl7.png)

可以看到整个组件都被涂成了黑色，我们还可以绘制更多好玩的图形：

```java
public void paint(Graphics g) {
    g.setColor(Color.BLACK);
    g.fillRect(0, 0, getWidth(), getHeight());
    g.setColor(Color.RED);    //画笔改成红色
  	//在中间画个圆角矩形边框
    g.drawRoundRect(getWidth() / 4, getHeight() / 4, getWidth() / 2, getHeight() / 2, 30, 30);
}
```

得到的效果就是这样了：

![image-20221028233307877](https://s2.loli.net/2022/10/28/xBsuijLa6beOwJy.png)

是不是感觉还挺好玩的，就像我们在玩画画游戏一样。这里列一下Graphics接口提供的所有功能：

```java
public abstract class Graphics {
   	//移动画笔原点到指定坐标，默认是(0,0)
    public abstract void translate(int x, int y);
    //设定画笔颜色
    public abstract void setColor(Color c);
    //设置为普通绘画模式
    public abstract void setPaintMode();
    //交替颜色模式，比较高级，小伙伴自行了解
    public abstract void setXORMode(Color c1);
    //设置字体，绘制文本内容时就按照这个字体来绘制
    public abstract void setFont(Font font);

    //设置裁剪区域，一旦设置裁剪区域，那么裁剪区域以外的地方即使绘制，也不会生效，绘制
  	//只会在裁剪区域内生效（有点像图层蒙版？）
    public abstract void setClip(int x, int y, int width, int height);
    //设定自定义形状的裁剪区域
    public abstract void setClip(Shape clip);

    //拷贝指定区域的内容到另一个位置
    public abstract void copyArea(int x, int y, int width, int height,
                                  int dx, int dy);
    //绘制直线
    public abstract void drawLine(int x1, int y1, int x2, int y2);
    //填充矩形区域
    public abstract void fillRect(int x, int y, int width, int height);
    //绘制矩形边框
    public void drawRect(int x, int y, int width, int height);
		//绘制圆角矩形边框
    public abstract void drawRoundRect(int x, int y, int width, int height,
                                       int arcWidth, int arcHeight);
    //填充圆角矩形区域
    public abstract void fillRoundRect(int x, int y, int width, int height,
                                       int arcWidth, int arcHeight);
    //绘制3D矩形边框（其实就是加了个深色和浅色边框，有一个视觉效果罢了）
    public void draw3DRect(int x, int y, int width, int height,
                           boolean raised);
    //填充3D矩形区域（同上）
    public void fill3DRect(int x, int y, int width, int height,
                           boolean raised);
    //绘制椭圆形边框
    public abstract void drawOval(int x, int y, int width, int height);
    //填充椭圆形区域
    public abstract void fillOval(int x, int y, int width, int height);
    //绘制弧线边框
    public abstract void drawArc(int x, int y, int width, int height,
                                 int startAngle, int arcAngle);
		//填充扇形区域
    public abstract void fillArc(int x, int y, int width, int height,
                                 int startAngle, int arcAngle);
    //绘制折线（需要提供多个坐标）
    public abstract void drawPolyline(int xPoints[], int yPoints[],
                                      int nPoints);
		//绘制多边形边框
    public abstract void drawPolygon(int xPoints[], int yPoints[],
                                     int nPoints);
    //填充多边形区域
    public abstract void fillPolygon(int xPoints[], int yPoints[],
                                     int nPoints);
    //绘制文本
    public abstract void drawString(String str, int x, int y);
   	//绘制图片（绘制大小为图片原本大小）
    public abstract boolean drawImage(Image img, int x, int y,
                                      ImageObserver observer);
   	//绘制按自定义大小缩放后的图片
    public abstract boolean drawImage(Image img, int x, int y,
                                      int width, int height,
                                      ImageObserver observer);
    //绘制图片时如果是透明部分则采用背景颜色填充
    public abstract boolean drawImage(Image img, int x, int y,
                                      Color bgcolor,
                                      ImageObserver observer);
    //绘制按自定义大小缩放后带背景颜色的图片
    public abstract boolean drawImage(Image img, int x, int y,
                                      int width, int height,
                                      Color bgcolor,
                                      ImageObserver observer);
    //对原本的图片按照起始坐标和尺寸进行裁剪后，再以给定大小绘制到给定位置
    public abstract boolean drawImage(Image img,
                                      int dx1, int dy1, int dx2, int dy2,
                                      int sx1, int sy1, int sx2, int sy2,
                                      ImageObserver observer);
    //累了
    public abstract boolean drawImage(Image img,
                                      int dx1, int dy1, int dx2, int dy2,
                                      int sx1, int sy1, int sx2, int sy2,
                                      Color bgcolor,
                                      ImageObserver observer);
}
```

我们这里要实现的时绘制一个图片，那么我们就可以像这样编写了：

```java
public class ImageView extends Component {
    private final Image image;   
    public ImageView(String filename) throws IOException {
        File file = new File(filename);
        image = ImageIO.read(file);   //我们可以使用ImageIO类来快速将图片文件读取为Image对象
    }

    @Override
    public void paint(Graphics g) {
      	//绘制图片需要提供Image对象
        g.drawImage(image, 0, 0, getWidth(), getHeight(), null);
    }
}
```

我们来试试看效果吧：

![image-20221028235756338](https://s2.loli.net/2022/10/28/5adDsGr2iRxywCX.png)

可以看到图片成功绘制出来了，这样，我们就提供自己编写绘制逻辑，成功完成了一个简单的自定义组件。

当然，现在我们讲了如何加载图片，顺便把设定自定义的程序图标介绍一下吧：

```java
Image image = ImageIO.read(new File("test.png"));
frame.setIconImage(image);
```

注意，在MacOS下这样写没用，得用专用的增强包：

```java
Image image = ImageIO.read(new File("test.png"));
Application.getApplication().setDockIconImage(image);
```

这样，我们的程序就会显示为我们自己定义的图标了。

### 窗口修饰和自定义形状

实际上我的窗口在默认情况下都是处于修饰状态，那么什么是修饰状态呢？

窗口修饰实际上就是我们窗口外面添加的边框：

![image-20221026164200924](https://s2.loli.net/2022/10/26/fMQDFCmyqIvJeTl.png)

有些时候，可能我们并不需要系统为我们提供的窗口边框，我们希望能够自己编写窗口的边框，包括各种按钮等，此时我们就可以将窗口设定为非修饰状态：

```java
public static void main(String[] args) throws IOException {
    Frame frame = new Frame("我是窗口");
    frame.setUndecorated(true);   //将窗口设定为非修饰状态
    frame.setSize(200, 200);
    frame.setVisible(true);
}
```

非修饰状态下，就只有一个窗口本身了：

![image-20221029111107959](https://s2.loli.net/2022/10/29/u9jSlmAc2GXr4VJ.png)

并且这个窗口是无法完成拖拽操作的，要实现拖拽还得我们自己编写（太原始了）可以看到，在默认情况下窗口的形状是一个方形的，我们可以将其调整为其他形状：

```java
public static void main(String[] args) throws IOException {
    Frame frame = new Frame("我是窗口");
    frame.setUndecorated(true);
    frame.setSize(200, 200);
  	//注意，只有窗口在非修饰状态下才能设定形状
  	//这里我们使用圆角矩形，形状最好跟窗口大小一样
    frame.setShape(new RoundRectangle2D.Double(0, 0, 200, 200, 20, 20));
    frame.setVisible(true);
}
```

可以看到，我们的窗口变成了这样：

![image-20221029111439062](https://s2.loli.net/2022/10/29/areQf2g3I74mlpV.png)

变成了好看的圆角矩形（但是这个圆角处理得不太好，有点毛毛糙糙的）圆角矩形也是现代操作系统窗口的设计语言。

我们也可以自行为窗口添加标题栏，同样只需要重写一下`paint`方法自行绘制就可以了：

```java
Frame frame = new Frame("我是窗口") {    //使用匿名内部类（或者自己写个子类也行）
    @Override
    public void paint(Graphics g) {
        g.setColor(Color.LIGHT_GRAY);
        g.fillRect(0, 0, getWidth(), 28);   //先绘制标题栏
        g.setColor(Color.BLACK); 
        g.drawString(getTitle(), getWidth() / 2, 20);   //绘制标题名称
        super.paint(g);   //原本的绘制别覆盖了，该怎么做还要怎么做
    }
};
```

我们来看看效果吧：

![image-20221029112035219](https://s2.loli.net/2022/10/29/hQ2YLjSgazM9Wkd.png)

是不是感觉不依靠操作系统，我们自己也能写一个好看的窗口出来了？

只不过这个窗口还不能拖动，我们来实现一下按住标题栏就可以拖动：

```java
frame.addMouseMotionListener(new MouseMotionAdapter() {   //只需要写一个监听器就可以搞定了
    int oldX, oldY;
    public void mouseDragged(MouseEvent e) {   //鼠标拖动时如果是标题栏，就将窗口位置修改
        if(e.getY() <= 28)
            frame.setLocation(e.getXOnScreen() - oldX, e.getYOnScreen() - oldY);
    }

    public void mouseMoved(MouseEvent e) {   //记录上一次的鼠标位置
        oldX = e.getX();
        oldY = e.getY();
    }
});
```

至此，有关AWT相关的内容，我们就讲解到这里，相信各位小伙伴肯定已经跃跃欲试想要开发一个自己的桌面应用程序了。只不过很遗憾，Java官方并没有再对AWT相关内容进行维护，因为AWT采用的是取不同操作系统交集策略，因为有些功能只有部分操作系统才有，这就导致很多功能都被砍掉，维护起来也很困难。下节课开始，我们会继续介绍Swing相关组件。

***

## Swing组件介绍

前面我们介绍了AWT，通过Java官方为我们提供的GUI框架，我们就可以编写出自己的桌面应用程序了，现在各位小伙伴应该已经有着良好的图形化界面开发基础了。

而Swing组件才是我们要学习的重点内容，它也是一套GUI框架，但是它是基于AWT编写的上层框架。

> Swing 是在AWT的基础上构建的一套新的图形界面系统，它提供了AWT 所能够提供的所有功能，并且用纯粹的Java代码对AWT 的功能进行了大幅度的扩充。例如说并不是所有的操作系统都提供了对树形控件的支持， Swing 利用了AWT 中所提供的基本作图方法对树形控件进行模拟。由于 Swing 控件是用100%的Java代码来实现的，因此在一个平台上设计的树形控件可以在其他平台上使用。由于在Swing 中没有使用本地方法来实现图形功能，我们通常把Swing控件称为轻量级控件。

其实简单来说，这玩意就是AWT那一套东西的扩展，或者说是强化版，很多东西还是沿用的AWT中的。

### 基本使用

我们来看看如何使用Swing编写桌面程序，首先还是最重要的窗口：

```java
public static void main(String[] args) {
    JFrame frame = new JFrame("我是窗口");   //Swing中的窗口叫做JFrame，对应的就是AWT中的Frame
    //它实际上就是Frame的子类，所以说我们之前怎么用的，现在怎么用就行了
    frame.setSize(500, 300);
    frame.setVisible(true);
}
```

是不是感觉学会AWT之后再看Swing也太简单了？

当然，既然是AWT的扩展，那肯定是有更多的新增功能的，比如我们之前想要实现点击X号关闭Java程序，这里我们只需要使用一个方法就可以设定了，不需要我们自己去写监听器：

```java
//我们可以直接为窗口设定关闭操作，JFrame已经为我们预设好了一些常用的操作了
frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);  //EXIT_ON_CLOSE就是直接退出程序，默认是只隐藏
```

Swing为我们提供了所有原本AWT中的组件的升级版，他们的名字前面都加上了J，比如按钮组件：

```java
public static void main(String[] args) {
    JFrame frame = new JFrame("我是窗口");
    frame.setLayout(null);
    JButton button = new JButton("Link Start");  //Button组件对应的就是JButton了
    button.setBounds(20, 50, 100, 50);
    frame.add(button);
    frame.setSize(500, 300);
    frame.setVisible(true);
}
```

Swing不像AWT那样，平台组件长啥样，就用什么，它的组件都是自行绘制的：

![image-20221029120313940](https://s2.loli.net/2022/10/29/dy6R2VNuOYIJamA.png)

这样，我们在不同的平台上，看到的组件UI样式，都会是一样的，不会出现长得不一样的情况。并且我们可以为组件自由替换皮肤，我们会在后面进行介绍。

还有，Swing在没有设定布局时，组件的坐标原点并不是窗口的左上角，而是窗口标题栏下方的左上角：

```java
JFrame frame = new JFrame("我是窗口");
frame.setSize(500, 300);
frame.setLayout(null);
frame.setDefaultCloseOperation(WindowConstants.EXIT_ON_CLOSE);

JButton button = new JButton("我是按钮");
button.setBounds(0, 0, 100, 30);

frame.add(button);
frame.setVisible(true);
```

![image-20221105171035570](https://s2.loli.net/2022/11/05/RMJWl576CiSDs8b.png)

这一点确实比AWT好很多，不然咱都不知道不同平台的标题栏到底是多高。至于为什么是这样，这是因为JFrame内部实际上单独维护了一个面板来存放组件，很多操作都被重定向给了内部的面板，这里就不深入说了，知道就行。

同样的，如果我们要使用菜单，直接使用对应的类即可：

```java
JFrame frame = new JFrame("我是窗口");
JMenuBar bar = new JMenuBar();    //JMenuBar对应的就是MenuBar
JMenu menu = new JMenu("我是菜单");
menu.add(new JMenuItem("选项1"));
menu.add(new JMenuItem("选项2"));
bar.add(menu);
frame.setJMenuBar(bar);
frame.setSize(500, 300);
frame.setVisible(true);
```

这个菜单也是Swing自己绘制的，如果是AWT的话，不同系统菜单位置还不一样，虽然这里是自己画的，但是效果看着还行：

![image-20221029120737715](https://s2.loli.net/2022/10/29/aMGHTu8sl2Sg4vm.png)

所以，Swing相关组件在使用上其实和我们之前学习的AWT是差不多的，只要前面AWT学的没问题，这里简直So Easy。

因为Swing是沿用的AWT框架体系，所以说AWT提供的旧组件，也是可以用的，但是这里不推荐：

```java
public static void main(String[] args) {
    JFrame frame = new JFrame("我是窗口");
    frame.setSize(500, 300);
    frame.add(new Button("我是按钮"));   //可以继续使用AWT组件，但是有新的咱肯定用新的啊
    frame.setVisible(true);
}
```

有关其他组件还请各位小伙伴自行了解。

### 新增组件介绍

Swing除了强化AWT提供的组件之外，还自行实现了各种各样新式的组件，我们来依次介绍一下。

首先是进度条组件：

![image-20221105163846233](https://s2.loli.net/2022/11/05/SY8wjEkdcArvxnQ.png)

很多时候我们都会用到进度条来展示某些任务的完成进度：

```java
JProgressBar bar = new JProgressBar();
bar.setMaximum(100);    //设定进度条的最大值
bar.setValue(50);    //设定进度值
bar.setBounds(20, 50, 100, 10);
```

我们可以利用进度条来写一个很简单的案例，比如文件的拷贝：

```java
JProgressBar bar = new JProgressBar();   //进度条显示文件拷贝进度
bar.setMaximum(1000);
bar.setBounds(20, 50, 300, 10);

JButton button = new JButton("点击开始");   //点击按钮开始拷贝文件
button.setBounds(20, 100, 100, 30);
button.addActionListener(e -> new Thread(() -> {
    //注意，不能直接在这个线程里面处理，因为这个线程是负责图形界面的，得单独创建一个线程处理，否则图形界面会卡死
    File file = new File("in.iso");
    try(FileInputStream in = new FileInputStream(file);
        FileOutputStream out = new FileOutputStream("out.iso")){
        long size = file.length(), current = 0;
        int len;
        byte[] bytes = new byte[1024];
        while ((len = in.read(bytes)) > 0) {
            current += len;
            bar.setValue((int) (bar.getMaximum() * (double)current / size));   //每次拷贝都更新进度条
          	bar.repaint();  //因为并不是每次更新值都会使得组件重新绘制，如果视觉上比较卡，可以每次拷贝都重新绘制组件
            out.write(bytes, 0, len);
        }
    } catch (IOException exception) {
        exception.printStackTrace();
    }
}).start());
```

这样，我们在拷贝文件的时候，就有一个进度条实时显示当前的进度了：

![image-20221105165756172](https://s2.loli.net/2022/11/05/qNoT6OwylH4Y8xc.png)

我们接着来看下一个组件，开关按钮：

```java
JToggleButton button = new JToggleButton("我是切换按钮");   //开关按钮有两个状态，一个是开一个是关
button.setBounds(20, 50, 100, 30);
```

它就像：

![image-20221105170052200](https://s2.loli.net/2022/11/05/1fBdjMOy4SnADHu.png)

它有着两个状态，我们点击一次会使得其切换到另一种状态：

![image-20221105170125623](https://s2.loli.net/2022/11/05/JXpw64yHb8rCRSg.png)

还有一些大型组件，比如颜色选择器：

```java
JColorChooser chooser = new JColorChooser();
chooser.setBounds(0, 0, 600, 300);
```

颜色选择器的主要作用顾名思义，就是让用户选择一个颜色：

![image-20221105170623359](https://s2.loli.net/2022/11/05/zsjxuSoYEr9lvZy.png)

这个太高级了，看着就很专业。同样的还有文件选择器JFileChooser：

![image-20221105170745920](https://s2.loli.net/2022/11/05/T6Vld1NMB9AJfct.png)

完了，这Mac越用咋越像Windows了。

当然，Swing考虑得不止这些，甚至连工具提示都有，啥是工具提示？

![image-20221105171336948](https://s2.loli.net/2022/11/05/mGaU6X2ILvqQT1g.png)

实际上就是当我们鼠标移动到某个组件上时，会给出一个漂浮提示，告诉我们这个组件是干嘛用的。

```java
JButton button = new JButton("我是按钮");
button.setBounds(50, 50, 100, 30);
button.setToolTipText("这个按钮是用来解决你毕设的！");
```

![image-20221105171453445](https://s2.loli.net/2022/11/05/izpvcCbVA3Ut81u.png)

`setToolTipText`方法是`JComponent`就带有的，因此任何组件都可以设置这样的工具提示，是不是感觉很高级？

还有文件树，我们经常在窗口中看到这样的：

![image-20221105171728838](https://s2.loli.net/2022/11/05/gcCnLterFaqPvkK.png)

我们的文件实际上在硬盘上就是以树形存储的，而Swing也为我们提供了能够显示树形关系的组件：

```java
JTree tree = new JTree();
tree.setBounds(0, 0, 200, 200);
```

![image-20221105171813979](https://s2.loli.net/2022/11/05/IvjYykGRXiOAMtK.png)

这样，我们就可以用它来做一个文件资源管理器了：

```java
//这里我们让JTree展示.idea目录下所有文件
File file = new File(".idea");   //这里我们列出.idea目录下所有文件
DefaultMutableTreeNode root = new DefaultMutableTreeNode(file.getName()); //既然是树形关系，肯定有一个根结点
//拿到当前目录下所有文件和文件夹
File[] files = Optional.ofNullable(file.listFiles()).orElseGet(() -> new File[0]);
for (File f : files)
    root.add(new DefaultMutableTreeNode(f.getName()));    //构造子结点并连接

JTree tree = new JTree(root);    //设定默认的根结点
tree.setBounds(0, 0, 200, 200);
```

来看看效果吧：

![image-20221105172802572](https://s2.loli.net/2022/11/05/YN4gKJbfstpRSCE.png)

你就说有没有内味吧。

### 多面板和分割面板

前面我们介绍了Swing为我们提供的丰富组件，我们接着来看多面板。

![image-20221105173345221](https://s2.loli.net/2022/11/05/keDg2pnsJolaUZH.png)

多面板顾名思义，就是为了在一个窗口中展示多个面板，但是面板是可以自由切换的，在顶部会有一个小小的标签，我们点击之后就可以切换到对应的面板了：

```java
JTabbedPane pane = new JTabbedPane();
pane.setBounds(0, 0, 500, 300);
pane.addTab("一号", new JPanel(){{setBackground(Color.ORANGE);}});
pane.addTab("二号", new JPanel(){{setBackground(Color.PINK);}});
```

JTabbedPane跟我们之前认识的Panel很像，相当于也是将我们的组件装进了内部，但是它可以同时装很多个，并且支持自由切换，所以说是很高级的。

这里我们创建两个面板，将一号面板设定为橙色，二号面板设定为粉色，分别添加到里面：

![image-20221105173821177](https://s2.loli.net/2022/11/05/4jp1N9LnmwJGtOl.png)

这样，我们就可以布置一号面板做某些事情，二号面板做另外一些事情了：

```java
JTabbedPane pane = new JTabbedPane();
pane.setBounds(0, 0, 500, 300); 
pane.addTab("一号", new JColorChooser());   //一号面板当颜色选择器
pane.addTab("二号", new JFileChooser());    //二号面板当文件选择器
```

高级感一下就出来了不是：

![image-20221105174105955](https://s2.loli.net/2022/11/05/QBPn8lk9tFd6sgH.png)

除了多面板这样的特殊面板组件之外，我们也可以使用分割面板：

![image-20221105174239436](https://s2.loli.net/2022/11/05/2gMxOrFGkHCJ73o.png)

分割面板将一块完整的面板分割为两个部分，这样，我们就可以分别在左右两边进行操作了，而且中间的分割线是可以拖动的，实际上我们的IDEA也是这样的：

![image-20221105174326135](https://s2.loli.net/2022/11/05/21l4GHg75fCaFzP.png)

IDEA的左边是文件管理器，右边就是编辑区域，同样支持拖动中间的分割线，这样的设计是非常人性化的。

我们来看看如何创建分割面板：

```java
JSplitPane pane = new JSplitPane();
pane.setOrientation(JSplitPane.HORIZONTAL_SPLIT);  //设定为横向分割
//横向分割之后，我们需要指定左右两边的组件
pane.setLeftComponent(new JPanel(){{setBackground(Color.ORANGE);}});
pane.setRightComponent(new JPanel(){{setBackground(Color.PINK);}});
```

![image-20221105174609500](https://s2.loli.net/2022/11/05/3VjoCckOZG2qNIf.png)

配合我们之前的JTree组件和JTextArea组件，我们也可以写一个简单的IDEA软件出来：

```java
JTextArea area = new JTextArea();   //右边就是我们需要编辑的文本域

File file = new File(".idea");
DefaultMutableTreeNode root = new DefaultMutableTreeNode(file);
File[] files = Optional.ofNullable(file.listFiles()).orElseGet(() -> new File[0]);
for (File f : files)
    root.add(new DefaultMutableTreeNode(f.getName()));
JTree tree = new JTree(root);   //左边就是我们的文件树
tree.addTreeSelectionListener(e -> {   //点击文件之后，我们需要变换编辑窗口中的文本内容，这里加个监听器
    area.setText("");   //先清空
    try (FileReader reader = new FileReader(".idea/"+e.getPath().getLastPathComponent().toString())){
        char[] chars = new char[128];   //直接开始读取内容
        int len;
        while ((len = reader.read(chars)) > 0)
            area.setText(area.getText() + new String(chars, 0, len));   //开始写入到编辑窗口中
    } catch (IOException ex) {
        ex.printStackTrace();
    }
});

pane.setLeftComponent(new JScrollPane(tree));   //文件树和编辑区域都套一个滚动面板，因为有可能会撑得很大
pane.setRightComponent(new JScrollPane(area));
```

我们来看看我们自己写的IDEA软件怎么样吧：

![image-20221105180609195](https://s2.loli.net/2022/11/05/rwik4EqaOeMYWfz.png)

嗯，真不错，各位小伙伴赶紧去JetBrains投简历吧！

### 选项窗口

前面我们介绍过对话框，但是AWT提供的对话框太过原始，很多功能都需要我们自行实现，而Swing为我们提供了一套已经实现好的预设选项对话框，我们只需要直接使用即可。

```java
JFrame frame = new JFrame("我是窗口");
frame.setSize(500, 300);
frame.setDefaultCloseOperation(WindowConstants.DO_NOTHING_ON_CLOSE);  //先将默认关闭行为设定为什么都不做
frame.addWindowListener(new WindowAdapter() {   //我们自己来实现窗口关闭行为
    @Override
    public void windowClosing(WindowEvent e) {   //这里我们可以直接展示一个预设好的确认对话框
        int value = JOptionPane.showConfirmDialog(frame, "你真的要退出吗？");
        if(value == JOptionPane.OK_OPTION)    //返回值就是用户的选择结果，也是预置好的，这里判断如果是OK那么就退出
            System.exit(0);
    }
});
```

我们之前要实现这样的一个功能，非常麻烦，但是现在就很简单了：

![image-20221106162732123](https://s2.loli.net/2022/11/06/ekOZLQnUR2dMXTN.png)

官方已经给我们预设好了一个对话框，我们直接用就可以了。当然，还有各种类型的，我们可以自己定义窗口的标题、图标等：

```java
JOptionPane.showConfirmDialog(frame, "你真的要退出吗？", "退出程序", JOptionPane.YES_NO_OPTION);
```

除了这种简单的对话框，Swing还为我们提供了一些其他类型的对话框，比如单纯的消息提示框：

```java
JOptionPane.showMessageDialog(frame, "我是简单的提示消息！");
```

![image-20221106165351473](https://s2.loli.net/2022/11/06/YH8dgDRunsG9jPv.png)

还有用户输入文本的输入对话框：

```java
JFrame frame = new JFrame("我是窗口");
frame.setSize(500, 300);
frame.setDefaultCloseOperation(WindowConstants.DO_NOTHING_ON_CLOSE);
frame.addWindowListener(new WindowAdapter() {
    @Override
    public void windowClosing(WindowEvent e) {
        String str = JOptionPane.showInputDialog("毕业后的你，将何去何从呢？");
        System.out.println(str);
    }
});
```

![image-20221106165324954](https://s2.loli.net/2022/11/06/ZwGfv1HqOjikEPn.png)

通过灵活使用这些对话框，用户与我们的交互就更加亲密了。

### 自定义主题

Swing早就考虑到了不同平台可能会出现的组件样式差异，因此推出了皮肤机制。

就像我们可以给英雄换皮肤一样，我们的组件UI也是可以换皮肤的，官方名称叫做LookAndFeel，Swing官方为我们提供了很多套皮肤，这些皮肤都是可以跨平台的，当然也有某些平台专属的限定皮肤：

* MetalLookAndFeel  -  官方默认皮肤
* WindowsLookAndFeel  -  Windows操作系统限定皮肤，其他平台无法使用
* MotifLookAndFeel   -   官方皮肤
* NimbusLookAndFeel   -   官方皮肤
* AquaLookAndFeel    -    MacOS操作系统限定皮肤，其他平台无法使用

更换皮肤很简单，我们只需要执行一个方法就可以，它是全局生效的：

```java
UIManager.setLookAndFeel(new AquaLookAndFeel());
```

这里我们将皮肤设定为MacOS的冰雪节限定皮肤AquaLookAndFeel：

![image-20221106170921703](https://s2.loli.net/2022/11/06/L7HyUlVpA5P9iTZ.png)

是不是感觉视觉上和之前的皮肤不太一样？我们可以多看看其他的皮肤：

![image-20221106171046755](https://s2.loli.net/2022/11/06/EGrWzIZuRfejXN2.png)

![image-20221106171110930](https://s2.loli.net/2022/11/06/BOtWrIe7CuklMcZ.png)

实际上Swing组件的绘制并不是由组件本身编写的，而是在各个UI实现类中编写的，所以说要修改组件样式只需要更换皮肤即可。

除了全局设定皮肤之外，我们也可以单独对某些组件设定皮肤，每个组件都有自己的`getUI`方法，这个方法就是获取当前组件使用的UI样式的：

```java
System.out.println(tree.getUI());
```

这里得到的是：

![image-20221106224348544](https://s2.loli.net/2022/11/06/2NChELXRkoqJGMQ.png)

我们可以自己编写一个UI样式来为组件进行设定：

```java
public static class TestJButtonUI extends ButtonUI {   //继承对应的UI父类
    @Override
    public void paint(Graphics g, JComponent c) {   //我们只需要重写对应UI的paint方法就可以了
        int width = c.getWidth(), height = c.getHeight();
        g.setColor(Color.BLACK);
        g.fillRect(0, 0, width, height);
        g.setColor(Color.WHITE);
        JButton button = (JButton) c;
        g.drawString(button.getText(), 0, 20);
    }
}
```

我们只需要使用set方法来设定即可：

```java
JButton button = new JButton("我是按钮");
button.setBounds(0, 0, 200, 30);
button.setUI(new TestJButtonUI());   //将UI设定为我们自己定义的即可
```

这样就换成我们自己的皮肤了：

![image-20221106231437842](https://s2.loli.net/2022/11/06/53kHx2zTZ7wtUfC.png)

各位小伙伴甚至可以编写一套自己的UI，并制作成一个LookAndFeel，这样我们写出来的程序就非常个性化了。

***

## 项目实战

前面我们已经学习了Swing的全部内容，最后我们还是来做一个小项目吧！

### Intellij IDEA Extreme

我们的目标是用IDEA写一个IDEA（当然不会太复杂，只需要实现基本功能就可以了）

需求分析：

* 支持创建项目、管理项目文件
* 支持对源代码文件的编辑
* 支持一键编译、运行

做Swing项目，什么五子棋、坦克大战都弱爆了，这里我们直接手撕一个IDEA出来。


————————————————
版权声明：本文为柏码知识库版权所有，禁止一切未经授权的转载、发布、出售等行为，违者将被追究法律责任。
原文链接：https://www.itbaima.cn/zh-CN/document/td5tgn04nqmkrryt