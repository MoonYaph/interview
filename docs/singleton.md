### 单例模式

#### 特点
- 私有构造函数
- 声明静态单例对象

#### 场景

- 当类只能有一个实例，而且使用者可以从一个众所周知的访问点访问它时
- 当这个唯一的实例应该是通过子类实例可扩展的

#### 优点
- 对唯一实例可以进行有效的受控访问
- 防止存储为一个实例的全局变量污染命名空间
- 可以在实例化方法中改变具体使用的实例
- 允许可变数目的实例。实例化的入口在一个地方，可以方便的控制存在实例的数量。

```js
class Singleton{
  private constructor(){}
  static instance: Singleton;
  static getInstance(): Signleton{
    if (!Singleton.instance){
      Singleton.instance = new Signleton();
    }
    return Signleton.instance;
  }
}

class TypeSingleton extends Singleton {}

class AdvanceSingleton {
  private constructor(){}
  static instance: Singleton;
  static getInstance(type: string): Singleton{
    if (!AdvanceSingleton.instance){
      if(type === 'type') {
        AdvanceSingleton.instance = new TypeSingleton()
      } else {
        AdvanceSingleton.instance = new Singleton()
      }
    }
    return AdvanceSingleton.instance;
  }
}
```