## Fragment 的使用
Fragment 是可以被包裹在多个不同 Activity 内的，同时一个 Activity 内可以包裹多个 Fragment  ，Activity 就如一个大的容器，它可以管理多个 Fragment 。所有 Activity 与Fragment 之间存在依赖关系。

## Activity与 Fragment 的通信方案

### 1. 借助 Bundle 对象通过 fragment.setArguments(Bundle bundle) 来传递参数
```Java
public static MainFragment newInstance(String param1) {
       MainFragment fragment = new MainFragment();
       //在Activity中创建Bundle数据包，并调用Fragment的setArguments(Bundle bundle)方法
       Bundle args = new Bundle();
       args.putString(PARAM1, param1);
       fragment.setArguments(args);
       return fragment;
}
@Override
public void onCreate(Bundle savedInstanceState) {
   super.onCreate(savedInstanceState);
   if (getArguments() != null) {
       //在Fragment中通过getArguments().getString()获取Activity传来的值
       mParam1 = getArguments().getString(PARAM1);
   }
}
```

### 2. Handler 方案
```Java
public class MainActivity extends FragmentActivity{
      //声明一个Handler
      public Handler mHandler = new Handler(){       
          @Override
           public void handleMessage(Message msg) {
                super.handleMessage(msg);
                //TODO 处理消息
           }
     }

     ...

   }

public class MainFragment extends Fragment{
      //保存Activity传递的handler
       private Handler mHandler;
       @Override
       public void onAttach(Activity activity) {
            super.onAttach(activity);
           //这个地方已经产生了耦合，若还有其他的activity，这个地方就得修改
            if(activity instance MainActivity){
                  mHandler =  ((MainActivity)activity).mHandler;
            }
       }
       ...相应的处理代码
 }

```

### 3. 利用回调接口函数传递
```Java
public class HostActivity extends Activity implements MainFragment.Iface {

    public void doSomething() {
        // Do something
    }
}


public class MainFragment extends Fragment {

    private Iface mIface;

    @Override
    public void onAttach(Activity activity) {
        super.onAttach(activity);
        try {
            mIface = (Iface) activity;
        } catch (ClassCastException e) {
            throw new ClassCastException(activity.toString() + " must implement Iface");
        }
    }

    private void invokeActivityToDoSomething() {
        Activity activity = getActivity();
        if (activity != null) {
            ((Iface) activity).doSomething();
        }
    }

    public interface Iface {
        void doSomething();
    }

}

```

### 4. 广播


### 5. EventBus
