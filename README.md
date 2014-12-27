Scopes
======

###What is Scopes?

Have you ever tried to set up scoped `ObjectGraphs` with Dagger and failed miserably? Scopes is here to help!

###What does Scopes do?
It allows to separate portions of your `Application` in logical "flows". It generates "`BaseActivity`s" that contain common dependencies that other `Activities` that are part of the same flow could use.

###What the hell are you talking about?!
Here is an example. Let's say that your `Application` has a login/signup flow (i.e. a screen with a login button, a another one with an "Enter username and password", etc). It is really likely that `Activities` that are part of this flow will have common dependencies (i.e. an `AuthenticatorService.java`,`LoginErrorDialog.java`, etc). Scopes allows you to define a `BaseActivity` that contains all these shared dependencies.

###Ok, got it, how do I use it?
It all starts by defining a class that is Annotated with `@Scope`; it does not need to be an `Activity`.

```java
@Scope(baseActivityName = "BaseLoginFlowActivity", retrofitServices = GithubService.class,
    restAdapterModule = RestAdapterModule.class, butterKnife = true)
public class LoginFlow {}
```

`baseActivityName` is the name you want to give to the parent `Activity` (it should be distinct for all `BaseActivities`)

`retrofitServices` takes in an array of `Class` `Objects` that are the retrofit `interfaces` you have defined

`restAdapterModule` is a `Module` that contains a provider for the `RestAdapter` to be used to create the `retrofitServices`

`butterKnife` it is an optional field that tells `Scopes` to wire up `ButterKnife` on the `BaseActivity`.
    
**Once you build your project**, `Scopes` will generate `BaseLoginFlowActivity.java` with the following content: 

```java
package scopes;

import android.app.Activity;
import android.os.Bundle;
import dagger.ObjectGraph;
import butterknife.ButterKnife;
public abstract class BaseLoginFlowActivity extends Activity {
  @javax.inject.Inject
  protected services.GithubService githubService;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(getLayout());
    ButterKnife.inject(getActivity());
    ObjectGraph.create(new scopes.BaseLoginFlowActivityModule()).plus(getModules()).inject(this);
  }
  protected abstract Object[] getModules();
  protected abstract Activity getActivity();
  protected abstract int getLayout();
}
```

As you can see `Scopes` creates `BaseLoginFlowActivityModule.java` that contains `@Providers` for the `retrofitServices`. This class uses the `RestAdapter` you provided to `@Scope` to create the `retrofitServices`.

```java
package scopes;

import services.GithubService;
import retrofit.RestAdapter;
@dagger.Module(injects = scopes.BaseLoginFlowActivity.class, includes = modules.RestAdapterModule.class)
public class BaseLoginFlowActivityModule {
  @dagger.Provides
  public GithubService providesGithubService(RestAdapter adapter) {
    return adapter.create(services.GithubService.class);
  }
}
```
    
### This is all fine an great, but what else do I have to do?
Now, you have to make your `Activity` extends the `BaseActivity` generated by `Scopes`; in this case it will be `BaseLoginFlowActivity`. You will have to implement a couple methods, in this case:

```java
protected abstract Object[] getModules();
protected abstract Activity getActivity();
protected abstract int getLayout();
```

Here is a basic example of a class that extends `BaseLoginFlowActivity`:

```java    
package me.emmano.scopes.app;

import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

import java.util.List;

import butterknife.InjectView;
import modules.ActivityModule;
import retrofit.Callback;
import retrofit.RetrofitError;
import retrofit.client.Response;
import scopes.BaseLoginFlowActivity;
import services.Repo;


public class MainActivity extends BaseLoginFlowActivity {

    @InjectView(R.id.text)
    protected TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        textView.setText("Eureka!");
        githubService.starGazers(new Callback<List<Repo>>() {
            @Override
            public void success(List<Repo> repos, Response response) {
                for (Repo repo : repos) {
                    Log.e(MainActivity.class.getSimpleName(), repo.getLogIn());
                }
            }

            @Override
            public void failure(RetrofitError error) {

            }
        });
    }

    @Override
    protected Object[] getModules() {
        return new Object[]{new ActivityModule()};
    }

    @Override
    protected Activity getActivity() {
        return this;
    }

    @Override
    protected int getLayout() {
        return R.layout.activity_main;
    }
}
```

There is one last thing for you to do. `getModules()` gives you the option to add extra dependencies to be used just in this class (namely, `MainActivity`). The most simplistic implementation of a `Module` could be as follows:

```java
package modules;

import dagger.Module;
import me.emmano.scopes.app.MainActivity;
import scopes.BaseLoginFlowActivityModule;

@Module(injects = MainActivity.class, addsTo = BaseLoginFlowActivityModule.class)
public class ActivityModule {}
```
    
Please note `addsTo`. Unfortunately, you will have to manually add the `Module` to which we are plus()sing. This will always be the `Module` generated by `Scope` that is injected on the `BaseActivity`; `BaseLoginFlowActivityModule` in this case.

### TODO
Upload an aar (that works!) to jcenter() so `Scopes` can be used by adding `compile...` to `build.gradle`

Create an Annotation that that can be added to a method in the `Application` `Class` to get a reference of an `Application` `ObjectGraph` if there is one.

Tons of refactoring. Kittens are currently dying due to some code on the `ScopeProcessor` class.

Add a parameter to `@Scope` that allows passing an `Classes[]` to be injected. Right now, only `Retrofit` services can be injected. You can currently add these dependencies to the your version of `ActivityModule`, add the corresponding `@Injects` and extends your version of `MainActivity` to get regular `Objects` other than `retrofitServices` injected. It is hacky and nasty, I know.

License
-------

    The MIT License (MIT)

    Copyright (c) 2014 Emmanuel Ortiguela

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:
    
    The above copyright notice and this permission notice shall be included in
    all copies or substantial portions of the Software.
    
    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
    THE SOFTWARE.
