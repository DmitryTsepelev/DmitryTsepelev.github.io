---
title: "Terminal–based game in 150 lines"
date: 2024-09-24 09:00:00 +0300
permalink: 'terminal-game'
description: 'Terminal–based game written in pure Ruby using less then 150 lines of code'
tags: ruby
---

In this article we will write a terminal–based real–time dungeon crawler. I will to keep it under 150 lines with idiomatic (no code golfing!) Ruby code.

![Result game](/assets/game.gif)

# Main game loop

I want to start with something basic. We need a class that will contain our game state and draw the screen. Let's create a `game.rb` with the following content:

```ruby
class Game
  SLEEP_INTERVAL = 0.2

  def run
    loop do
      draw_screen
      sleep SLEEP_INTERVAL
    end
  end

  private

  def draw_screen
    system "clear"

    puts Time.now
  end
end

Game.new.run
```

> Source is [here](https://gist.github.com/DmitryTsepelev/1e9d73db26aa19d4ad2bd6ddbd67f045#file-1_game-rb)

Run it (using `ruby ./game.rb`) and you see that it updates the current time every `SLEEP_INTERVAL` seconds. Nothing fancy yet!

# Rendering the map

Our next goal is to render a level map. Let's keep the map in the file called `map.txt`, which will be in the same directory as a `game.rb`. We need five types of objects:

- an empty place (`s`);
- a start point—a place where player appears (`p`);
- a final point—an exit from the level (`d`);
- a tree—the cell that could not be entered (`t`);
- an enemy—something that kills the player when they appear on the same place (`e`).

Feel free to design you own map, or use mine:

```
tsssstsdst
tsssssesst
tstssssset
tstsssssst
tsstssssst
tssssstsst
tstssstsst
tpstsstsst
```

Here is a class that parses the file and renders trees and empty spaces:

```ruby
TREE, SPACE = '🌲', "・"

Level = Struct.new(:map, :enemies, :player, :door)

class LevelBuilder
  def initialize(filepath)
    @filepath = filepath
  end

  MAPPING = { 't' => TREE, 's' => SPACE }

  def build
    Level.new.tap do |level|
      level.enemies = []

      level.map = File.readlines(@filepath).map do |line|
        line.sub("\n", "").chars.map do |c|
          MAPPING[c] || SPACE
        end
      end
    end
  end
end
```

Note that we do not handle start location, final location and enemies yet. Instead, we treat them as empty spaces: since we will have a specific logic tied to them, we will add it later.

Now we need to update our game class to draw (and redraw) the field:

```ruby
class Game
  SLEEP_INTERVAL = 0.2

  def run
    @level = LevelBuilder.new("./map.txt").build

    loop do
      draw_screen
      sleep SLEEP_INTERVAL
    end
  end

  private

  def draw_screen
    system "clear"

    @level.each do |row|
      row.each do |cell|
        print cell
      end
      puts
    end
  end
end
```

The only thing it does is just prints whatever we have in in the `map` field. The only thing that might be unfamiliar to you is `system "clear"`, which just cleans up the terminal.

> Source is [here](https://gist.github.com/DmitryTsepelev/1e9d73db26aa19d4ad2bd6ddbd67f045#file-2_game-rb)

# Adding complex objects

Now we need to render player, enemies and door. Let's extend our `Level` class definition to support them and decide how we will draw them on the screen:

```ruby
PLAYER, ENEMY, DOOR, TREE, SPACE = '🧙', '👻', '🚪', '🌲', "・"

Level = Struct.new(:map, :enemies, :player, :door)
```

After that, let's define another struct called `DynamicObject` to represent them:

```ruby
DynamicObject = Struct.new(:row_idx, :col_idx, :kind)
```

Now we need to adjust our `LevelBuilder` to handle new objects. Note that when we find one of those we assign them to separate fields and treat them as empty spaces on the map, because they can change their location:

```ruby
class LevelBuilder
  def initialize(filepath)
    @filepath = filepath
  end

  MAPPING = { 't' => TREE, 's' => SPACE }

  def build
    Level.new.tap do |level|
      level.enemies = []

      level.map = File.readlines(@filepath).map.with_index do |line, row_idx|
        line.chars.map.with_index do |c, col_idx|
          case c
          when 'e'
            level.enemies << DynamicObject.new(row_idx, col_idx, :enemy)
            SPACE
          when 'p'
            level.player = DynamicObject.new(row_idx, col_idx, :player)
            SPACE
          when 'd'
            level.door = DynamicObject.new(row_idx, col_idx, :door)
            SPACE
          else
            MAPPING[c]
          end
        end
      end
    end
  end
end
```

Finally, we need to update `Game#draw_screen` to support our new primitives:

```ruby
def draw_screen
  system "clear"

  @level.map.each_with_index do |row, row_idx|
    row.each_with_index do |cell, col_idx|
      if @level.player.row_idx == row_idx && @level.player.col_idx == col_idx
        print PLAYER
      elsif @level.door.row_idx == row_idx && @level.door.col_idx == col_idx
        print DOOR
      elsif @level.enemies.find { |enemy| enemy.row_idx == row_idx && enemy.col_idx == col_idx }
        print ENEMY
      else
        print cell
      end
    end
    puts "\n"
  end
end
```

> You can find the whole snippet [here](https://gist.github.com/DmitryTsepelev/1e9d73db26aa19d4ad2bd6ddbd67f045#file-3_game-rb)

# How to read user input

Our next goal is to find a way to control the character. We want to be able to move it up, down, left and right. Let's start with the code that checks if any key is pressed right now.

```ruby
class Game
  # ...

  private

  def get_pressed_key
    begin
      system('stty raw -echo')
      (STDIN.read_nonblock(4).ord rescue nil)
    ensure
      system('stty -raw echo')
    end
  end
end
```

This looks like a bit cryptic, so we will examine it line by line:

- `stty raw -echo` turns off echoing of input keystrokes;
- `STDIN.read_nonblock(4)` reads 4 bytes from user input, and `.ord` returns the integer representation of the entered character (e.g., `'w'.ord == 19`);
- `stty -raw echo` turns echoing back on.

Let's call this method inside the game loop and use the value to update player's position:

```ruby
class Game
  # ...

  def run
    @level = LevelBuilder.new("./map.txt").build

    loop do
      draw_screen

      new_player_position = DynamicObject.new(@level.player.row_idx, @level.player.col_idx)
      new_player_position.move(get_pressed_key)

      @level.player = new_player_position

      sleep SLEEP_INTERVAL
    end
  end

  # ...
end
```

Finally, we have to implement `DynamicObject#move`. Let's use `WASD` keys to control movements:

```ruby
UP, DOWN, RIGHT, LEFT = 119, 115, 100, 97

DynamicObject = Struct.new(:row_idx, :col_idx, :kind) do
  def move(dir)
    case dir
    when RIGHT then self.col_idx += 1
    when LEFT then self.col_idx -= 1
    when UP then self.row_idx -= 1
    when DOWN then self.row_idx += 1
    end
  end
end
```

> Source is [here](https://gist.github.com/DmitryTsepelev/1e9d73db26aa19d4ad2bd6ddbd67f045#file-4_game-rb)

Try it out! You can see that player position is updated on the screen.

> If game feels a bit slow to you—try to play with `SLEEP_INTERVAL` value.

# Collision handling

If you ran the previous example, you might have noticed that player can step on trees, ghosts, door and even leave the screen. The reason is that we update the position regardless of what is placed under it. Let's fix that!

First of all, we need to add a method that will check new position and tell us what will happen if we move object there. Note that we will make this method generic: it will be able to check if passed object collides with any of objects we pass. We will use it to handle other collisions later.

There can be multiple outcomes:

- `:out_of_border` is returned when object attempts to leave the border of the level;
- `:tree`, `:ghost`, `:door` or `:player` is returned when object bumps into the corresponding object.

This is how `Game#run` looks like:

```ruby
class Game
  # ...

  def run
    @level = LevelBuilder.new("./map.txt").build

    loop do
      draw_screen

      new_player_position = DynamicObject.new(@level.player.row_idx, @level.player.col_idx)
      new_player_position.move(get_pressed_key)

      case check_collision(new_player_position.row_idx, new_player_position.col_idx, @level.enemies + [@level.door])
      when :door
        puts "🎉 Level passed 🎉"
        break
      when :enemy
        puts "☠️ You died ☠️"
        break
      when nil
        @level.player = new_player_position
      end

      sleep SLEEP_INTERVAL
    end
  end

  # ...
end
```

Now we should implement `Game#check_collision`:

```ruby
class Game
  # ...

  private

  def check_collision(row_idx, col_idx, objects)
    return :out_of_border if row_idx < 0 || row_idx >= @level.map.length || col_idx < 0 || col_idx >= @level.map[0].length
    return :tree if @level.map[row_idx][col_idx] == TREE
    objects.find { _1.row_idx == row_idx && _1.col_idx == col_idx }&.kind
  end
  # ...
end
```

> Source is [here](https://gist.github.com/DmitryTsepelev/1e9d73db26aa19d4ad2bd6ddbd67f045#file-5_game-rb)

Try to run the game and see that now we handle collisions properly: there's no way to cross the border, step on the tree and game ends when you bump into the ghost or door.

# Adding fancy AI to our ghosts

Our ghosts are not dangerous at all, because they can't move, let's address that! First of all, we need to change our main loop to trigger movement. Moreover, we should check the collisions after that to make sure that ghost did not bump into the player:

```ruby
class Game
  # ...

  def run
    @level = LevelBuilder.new("./map.txt").build

    loop do
      move_enemies
      draw_screen

      case check_collision(@level.player.row_idx, @level.player.col_idx, @level.enemies)
      when :enemy
        puts "☠️ You died ☠️"
        break
      end

      new_player_position = DynamicObject.new(@level.player.row_idx, @level.player.col_idx)
      new_player_position.move(get_pressed_key)

      case check_collision(new_player_position.row_idx, new_player_position.col_idx, @level.enemies + [@level.door])
      when :door
        puts "🎉 Level passed 🎉"
        break
      when :enemy
        puts "☠️ You died ☠️"
        break
      when nil
        @level.player = new_player_position
      end

      sleep SLEEP_INTERVAL
    end
  end

  # ...
end
```

Now we can implement the smart AI–powered algorithm that makes decision on enemy movement:

```ruby
class Game
  # ...

  def move_enemies
    @level.enemies.each_with_index do |enemy, idx|
      next if rand(1) > 0.8

      new_enemy = DynamicObject.new(enemy.row_idx, enemy.col_idx, :enemy)
      new_enemy.move([RIGHT, LEFT, UP, DOWN].sample)
      @level.enemies[idx] = new_enemy if check_collision(new_enemy.row_idx, new_enemy.col_idx, [@level.door, @level.player]).nil?
    end
  end

  # ...
```

> Source is [here](https://gist.github.com/DmitryTsepelev/1e9d73db26aa19d4ad2bd6ddbd67f045#file-6_game-rb)

# Extracting the rendering logic

I have a feeling that our `Game` class becomes some kind of a [God object](https://en.wikipedia.org/wiki/God_object): it handles user input, screen rendering and movement logic. Let's try to extract all rendering–related code to the separate class:

```ruby
class Screen
  def render_level(level)
    system "clear"

    level.map.each_with_index do |row, row_idx|
      row.each_with_index do |cell, col_idx|
        if level.player.row_idx == row_idx && level.player.col_idx == col_idx
          print PLAYER
        elsif level.door.row_idx == row_idx && level.door.col_idx == col_idx
          print DOOR
        elsif level.enemies.find { |enemy| enemy.row_idx == row_idx && enemy.col_idx == col_idx }
          print ENEMY
        else
          print cell
        end
      end
      puts "\n"
    end
  end

  def render_death_message = puts "☠️ You died ☠️"
  def render_level_passed_message = puts "🎉 Level passed 🎉"
end
```

With this change we can easily replace terminal with a web page or even print game state to the paper!

Now we can change the main loop `Game` class and drop `#draw_screen` method:

```ruby
class Game
  # ...

  def run
    screen = Screen.new

    @level = LevelBuilder.new("./map.txt").build

    loop do
      move_enemies
      screen.render_level(@level)

      case check_collision(@level.player.row_idx, @level.player.col_idx, @level.enemies)
      when :enemy
        screen.render_death_message
        break
      end

      new_player_position = DynamicObject.new(@level.player.row_idx, @level.player.col_idx)
      new_player_position.move(get_pressed_key)

      case check_collision(new_player_position.row_idx, new_player_position.col_idx, @level.enemies + [@level.door])
      when :door
        screen.render_level_passed_message
        break
      when :enemy
        screen.render_death_message
        break
      when nil
        @level.player = new_player_position
      end

      sleep SLEEP_INTERVAL
    end
  end

  # ...
end
```

> Final code is [here](https://gist.github.com/DmitryTsepelev/1e9d73db26aa19d4ad2bd6ddbd67f045#file-7_game-rb)

---

Let's take a break here: we built more or less playable game in 138 lines. I have some more ideas for you to practice:

- try to add fireballs that are created when player presses `SPACE` and which should eliminate ghosts;
- try to add more static objects (like houses 🏠) to the level—you will probably need to refactor `LevelBuilder#build` to some kind of DSL;
- add a way to have more than one level (probably you will need to turn `Game` class into the state machine);
- build up level generator to make the game endless.
