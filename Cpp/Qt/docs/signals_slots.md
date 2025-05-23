## QtCreator 直接转到槽

* 在 QtCreator 中添加槽很简单，直接在设计界面中右键 Button 选择 `转到槽`，再选择信号即可，QtCreator 会自动给对应界面的头文件的类中添加槽函数声明，并在 cpp 文件中添加函数体

## VS 手动添加槽函数

* 在 VS 中不会自动完成这些操作，手动完成即可。先在头文件中类的 `private slots` 作用域中添加槽函数声明，再在 cpp 文件中添加实现
* 随后打开 ui 界面，添加一个 PushButton 控件，按下 F4，拖动 Button 拉出 `配置连接` 选项界面
* 点击右侧的编辑会弹出新窗口，点击上侧的加号，手动添加需要响应的槽函数即可
* 保存后关闭设计界面，给槽函数添加需要实现的功能，运行后点击按钮即可执行槽函数

## 通过代码手动连接信号槽

* 通过代码完成效率更高，无需打开设计师界面即可完成所需功能
* 在需要操作的界面的构造函数中添加控件，然后使用 connect 函数连接已声明和实现的槽函数即可

```cpp
dlg::dlg(QWidget* parent) : QMainWindow(parent), ui(new Ui::dlg) {
  ui->setupUi(this);

  QPushButton* btn = new QPushButton();
  btn->setText("run slot");

  connect(btn, &QPushButton::clicked, this, &dlg::slot);
}

void dlg::slot() {
  QLabel* label = new QLabel();
  label->setText("show");
  label->show();
}
```

* connect 有五种重载形式

```cpp
// Qt4 的写法，用 SIGNAL() 和 SLOT() 两个宏将信号和槽转为 const char*
QMetaObject::Connection connect(const QObject *, const char *, const QObject *,
                                const char *, Qt::ConnectionType);
// QMetaMethod 是 Qt 中一个能获取函数详细信息的类，也是 Qt4 的写法
QMetaObject::Connection connect(const QObject *, const QMetaMethod &,
                                const QObject *, const QMetaMethod &,
                                Qt::ConnectionType);
// 省略接受者，实际将 this 设为接受者
QMetaObject::Connection connect(const QObject *, const char *, const char *,
                                Qt::ConnectionType) const;
// Qt5 写法，信号和槽可以是成员函数指针
QMetaObject::Connection connect(const QObject *, PointerToMemberFunction,
                                const QObject *, PointerToMemberFunction,
                                Qt::ConnectionType);
// 省略接受者，不需要 this，第三个参数也可以是一个 lambda、static 函数、全局函数
QMetaObject::Connection connect(const QObject *, PointerToMemberFunction,
                                Functor);
```

## 通过自定义信号为槽函数传参

* 通常使用的信号由 Qt 控件自带，而槽函数的参数必须比信号的参数少，原生信号没有参数（实际上 QPushButton::clicked 有一个参数，但不能自己指定），因此槽函数也不能指定参数

```cpp
connect(btn, &QPushButton::clicked, this, &dlg::slot);  // slot 不能有参数
```

* 要实现传参，需要自定义信号。信号在头文件 signals 作用域中声明，没有返回值，也不需要在源文件中实现（实际上为了通过编译，moc 会自动添加函数体）

```cpp
class dlg : public QMainWindow {
  Q_OBJECT

 public:
  explicit dlg(QWidget* parent = 0);
  ~dlg();

 private:
  Ui::Form* ui;
  QPushButton* btn;

 signals:
  void send(int n);  // 自定义信号

 private slots:
  void rcv(int n);  // 用于接受自定义信号的槽函数
  void clk();       // 连接给 btn 的无参数槽函数
};
```

* 连接自定义信号后，使用 emit 即可触发所有连接到此信号的槽函数

```cpp
connect(this, &dlg::send, this, &dlg::rcv);
emit send(3);  // 触发槽函数

void widget::rcv(int n) { btn->setText(QString::number(n)); }
```

* 因此如果希望使用原生信号，又想传参给槽函数，只要在无参的槽函数中发送信号，即可触发有参数版本的槽函数

```cpp
connect(btn, &QPushButton::clicked, this, &dlg::clk);

void dlg::clk() {
  emit send(4);  // 传递参数 4
}
```

* 信号槽不是 GUI 提供的，在 Qt 控制台程序中也可以使用，只需要让类继承 QObject 即可

## 父子窗口来回切换

* 在父窗口中设置一个按钮，点击将隐藏父窗口，创建并显示一个子窗口。在子窗口设置一个返回按钮，点击将关闭子窗口，并重新显示隐藏的父窗口

```cpp
// parent.h
#include "child.h"

class parent {
 private slots:
  void clickBtn();
  void reshow();
};

// parent.cpp
void parent::clickBtn() {
  this->hide();
  child* w = new child;
  connect(w, &child::reshowSignal, this, &parent::reshow);
  w->show();
}

void parent::reshow() { this->show(); }

// child.h
#include "parent.h"
class child {
 signals:
  void reshowSignal();
 private slots:
  void goBack();
};

// child.cpp
void child::goBack() {
  emit reshowSignal();
  this->close();
}
```
