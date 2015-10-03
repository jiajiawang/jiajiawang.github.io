---
layout: post
comments: true
title:  "Experiment with Motion Game"
date:   2015-10-02 18:15
categories: rubymotion
---

[RubyMotion](http://www.rubymotion.com/) 4.0 was recently released with several great features.

The first one must be RubyMotion Starter, a free version of RubyMotion that
comes with full support for iOS and Android development. It's the right time to
get your hands on it if you are a ruby developer who wants to learn mobile
app development.

Another great feature is the introduction of
[motion-game](http://www.rubymotion.com/developers/motion-game/),
a brand-new, 100% cross-platform engine to write mobile games.
Ruby developer can now write games for iOS and Android with a single code base.
And this is what I'm going to do in the next.

Today, we will create a MarioJump game in 100 lines.


![Mario Jump]({{ site.url }}/assets/mario_jump.png)

Resources and source code can be downloaded from
[here](https://github.com/jiajiawang/MarioJump).

### Initialization

Let's initialize the game with command:

~~~
motion create --template=motion-game MarioJump
~~~

To see what it creates, execute `rake ios:simulator` (or `rake android:emulator`).

You'll see a black screen with nothing but a 'Hello World' label.

In this tutorial, the only file we will care about is 'app/main_scene.rb'.

### Add Mario & Ground

Let's remove the label and add our main character Mario.

~~~ruby
  class MainScene < MG::Scene
    def initialize
      add_mario
    end

    def add_mario
      @mario = MG::Sprite.new('mario_1.png')
      @mario.position = [200, 200]
      @mario.attach_physics_box
      add @mario
    end
  end
~~~

If you run the game now, you'll see Mario falls out of the screen soon. That's
because of the gravity. Let's tweak it and add a ground.

~~~ruby
  def initialize
    self.gravity = [0, -900]

    add_mario
    add_ground
  end

  def add_ground
    1.upto(15) do |offset|
      block = MG::Sprite.new('block.png')
      block.attach_physics_box
      block.dynamic = false
      block.position = [200 + (offset-1) * 48, 100]
      add block
    end
  end
~~~

### Run

Now, we can see Mario falls from the sky and then stands on the ground.

To let Mario run, update `add_ground`:

~~~ruby
  add block
  block.move_by([-1500, 0], 5.0) { block.delete_from_parent  }
  ...
~~~

We are actually moving the ground. But it looks like that Mario is moving.
And to make it look better, we can add an animation:

~~~ruby
  @mario.animate(['mario_1.png', 'mario_2.png', 'mario_3.png'], 0.1, :forever)
~~~

### Jump

To let Mario jump when you touch the screen, we can use:

~~~ruby
  on_touch_begin do
    @mario.velocity = [0, 400]
  end
~~~

However, by just doing this, Mario will jump whenever you touch the screen, even it's in the air.
To fix this, we'll need a variable `@on_ground` and event `on_contact_begin`.

~~~ruby
  class MainScene < MG::Scene
    MARIO = 1
    GROUND = 2

    def initialize
      ...

      @on_ground = false

      on_touch_begin do
        if @on_ground
          @on_ground = false
          @mario.velocity = [0, 400]
        end
      end

      on_contact_begin do
        @on_ground = true
        @mario.velocity = [0, 0]
        true
      end
    end

    def add_ground
      ...
        block.category_mask = GROUND
        block.contact_mask = MARIO
      ...
    end

    def add_mario
      ...
      @mario.category_mask = MARIO
      @mario.contact_mask = GROUND
      ...
    end
  end
~~~

After doing this, Mario can only jump when he stands on the ground.

### Build the level

Now it's time to build the level.

We'll be keeping adding endless moving grounds so that players can continuously
play the game if they don't fall from it.

~~~ruby
  def initialize
    ...
    @block_update = 0
    @random = Random.new

    start_update
  end

  def add_block
    number = @random.rand(1..5)
    1.upto(number) do |offset|
      block = MG::Sprite.new('block.png')
      block.attach_physics_box
      block.dynamic = false
      block.category_mask = GROUND
      block.contact_mask = MARIO
      block.position = [MG::Director.shared.size.width + offset * 48, 100]
      add block
      block.move_by([-1500, 0], 5.0) { block.delete_from_parent  }
    end
  end

  def update(delta)
    @block_update += delta
    if @block_update >= 1.0
      add_block
      @block_update = 0
    end
  end
~~~

This will create a 1~5 blocks long moving ground every one second.

### What's next

A featured game should have sounds, Game Over and Restart Game. To do it:

~~~ruby
@@ -11,9 +11,14 @@ class MainScene < MG::Scene
     @on_ground = false
     @block_update = 0
     @random = Random.new
+    @game_over = false

     on_touch_begin do
+      if @game_over
+        MG::Director.shared.replace(MainScene.new)
+      end
       if @on_ground
+        MG::Audio.play('jump.wav')
         @on_ground = false
         @mario.velocity = [0, 400]
       end
@@ -66,6 +71,12 @@ class MainScene < MG::Scene
   end

   def update(delta)
+    if @mario.position.y < 0
+      MG::Audio.play('game_over.wav')
+      stop_update
+      @game_over = true
+    end
~~~

We can now play the game. Survive as long as you can!
