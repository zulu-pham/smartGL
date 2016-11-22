#SmartGL for Android#

**SmartGL** is an Android Studio library that simplifies the use of **OpenGL** to display **2D Sprites** and **3D Textured Objects**.

It has been used by several games and apps.

![alt text](extras/smartgl_screenshot.png?raw=true "Screenshot example")

##Usage##

The scene is displayed inside a **SmartGLView**.

A **SmartGLViewController** interface is used to handle the scene (like adding objects to the scene).

A default **SmartGLRenderer** is used to render the scene in the view.


```xml
    <fr.arnaudguyon.smartgl.opengl.SmartGLView
        android:id="@+id/smartGLView"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
```

You can create an instance of SmartGLViewController or just implement it in your Activity or Fragment.

```java
public class MainActivity extends AppCompatActivity implements SmartGLViewController {

    private SmartGLView mSmartGLView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_main);

        mSmartGLView = (SmartGLView) findViewById(R.id.smartGLView);
        mSmartGLView.setDefaultRenderer(this);
        mSmartGLView.setController(this);
    }
}
```

When the SmartGLView is ready, **onPrepareView(...)** is called on the SmartGLViewController. This method is the best place to prepare the scene (load textures, and add objects to the scene).

When the SmartGLView is dismissed, **onReleaseView(...)** is called. This is the best place to release content from memory (like textures).

If the View is resized for any reason (rotation of the device for example), **onResizeView(...)** is called.

At every OpenGL frame, **onTick(...)** is called.

Finally, every touch interaction calls **onTouchEvent(...)**

```java
public class MainActivity extends AppCompatActivity implements SmartGLViewController {

    @Override
    public void onPrepareView(SmartGLView smartGLView) {
    }

    @Override
    public void onReleaseView(SmartGLView smartGLView) {
    }

    @Override
    public void onResizeView(SmartGLView smartGLView) {
    }

    @Override
    public void onTick(SmartGLView smartGLView) {
    }

    @Override
    public void onTouchEvent(SmartGLView smartGLView, TouchHelperEvent event) {
    }
}
```

###LifeCycle###

It is really important to inform the SmartGLView when the Activity or Fragment is paused or resumed. **If you miss this step the scene will not be initialized or restored correctly**.

```java
public class MainActivity extends AppCompatActivity implements SmartGLViewController {
    
    @Override
    protected void onPause() {
        if (mSmartGLView != null) {
            mSmartGLView.onPause();
        }
        super.onPause();
    }

    @Override
    protected void onResume() {
        super.onResume();
        if (mSmartGLView != null) {
            mSmartGLView.onResume();
        }
    }
}
```

###Preparing the scene###

The draw pipeline includes Render passes. For example you can have 3D objects in the background pass, and Sprites in the forground pass.

Let's start with a simple **Sprite** using a **Texture** (picture) loaded from the Android drawables. It is highly recommended to share the same texture between several Sprites or Objects 3D when possible: the textures are loaded into the GPU memory and it is often not that big.

```java
public class MainActivity extends AppCompatActivity implements SmartGLViewController {

    private Texture mSpriteTexture;
    
    @Override
    public void onPrepareView(SmartGLView smartGLView) {

        SmartGLRenderer renderer = smartGLView.getSmartGLRenderer();
        RenderPassSprite renderPassSprite = new RenderPassSprite();
        renderer.addRenderPass(renderPassSprite);  // add it only once for all Sprites

        mSpriteTexture = new Texture(context, R.drawable.planet);

        mSprite = new Sprite(120, 120);	// 120 x 120 pixels
        mSprite.setPivot(0.5f, 0.5f);  // position / rotation axis in the middle of the sprite
        mSprite.setPos(60, 60);
        mSprite.setTexture(mSpriteTexture);
        renderPassSprite.addSprite(mSprite);
    }

    @Override
    public void onReleaseView(SmartGLView smartGLView) {
        if (mSpriteTexture != null) {
            mSpriteTexture.release();
        }
    }

}
```

**Congratulations!** The Sprite is displayed on your screen!

Let's add some basic moves.

Framerate vary on devices. This is why you should always take it into account when moving elements on screen.

```java
public class MainActivity extends AppCompatActivity implements SmartGLViewController {

    @Override
    public void onTick(SmartGLView smartGLView) {

        SmartGLRenderer renderer = smartGLView.getSmartGLRenderer();
        float frameDuration = renderer.getFrameDuration();
        if (mSprite != null) {
            float newX = mSprite.getPosX() + (frameDuration * 100);
            float newY = mSprite.getPosY();
            if (newX > 600) {
                newX = 0;
            }
            mSprite.setPos(newX, newY);

            float newRot = mSprite.getRotation() + (frameDuration * 100);
            mSprite.setRotation(newRot);
    }

}
```

Now let's add a moving 3D Object! There is a **WavefrontModel** that can load Wavefront models from an .obj file.

See [Wave Front format on Wikipedia](https://en.wikipedia.org/wiki/Wavefront_.obj_file).

The WavefrontModel.Builder is used to load the model and convert it into a Object3D. For each material used in the model, you will have to assign a corresponding Texture (see AddTexture).

There is no standard unity for the size of the models, so when you load different models you'll probably have to scale them so that they look the correct size compared to each other. You can use **setScale()** for that.
The Camera is by default at (0,0,0) and looking to -Z direction. If you want your objects visible for the camera, you need to put them in front of it, using something like **setPos(0,0,-10)**.

TODO: describe Axis & Camera

```java
public class MainActivity extends AppCompatActivity implements SmartGLViewController {

    private Object3D mSpaceship;
    private Texture mShipTexture;

    @Override
    public void onPrepareView(SmartGLView smartGLView) {

        // ...

        RenderPassObject3D renderPassObject3D = new RenderPassObject3D();
        renderer.addRenderPass(renderPassObject3D);  // add it only once for all 3D Objects
        
        mShipTexture = new Texture(context, R.drawable.ship_picture);

        WavefrontModel model = new WavefrontModel.Builder(context, R.raw.cube_obj)
                .addTexture("Material001", mShipTexture)
                .create();
        mSpaceship = model.toObject3D();
        mSpaceship.setScale(0.1f, 0.1f, 0.1f);  // Adjust the scale if object is too big / too small
        mSpaceship.setPos(0, 0, -5);            // move the object behind the camera
        renderPassObject3D.addObject(mSpaceship);
    }

    @Override
    public void onTick(SmartGLView smartGLView) {

        // ...
        if (mSpaceship != null) {
            float rx = mSpaceship.getRotX() + 100*frameDuration;
            float ry = mSpaceship.getRotY() + 77*frameDuration;
            float rz = mSpaceship.getRotZ() + 56*frameDuration;
            mSpaceship.setRotation(rx, ry, rz);
        }
    
        @Override
        public void onReleaseView(SmartGLView smartGLView) {
            if (mShipTexture != null) {
                mShipTexture.release();
            }
        }
    }

}
```

TODO:

* find a free textured model
* create VBO for loaded Objects
* default texture VS color shader...
* advanced use documentation: build objects, other shaders, ...
* licence in the sources