---
layout: post
title: "Sudoku Solver AR Part 2: Sudoku"
date: 2017-07-27 10:32:00 -0500
---

_This post is part of a series where I explain how to build an **augmented reality Sudoku solver**. All of the image processing and computer vision is done from scratch. See the [first post]({% post_url 2017-07-20-sudoku-solver-ar-part-1-about %}) for a complete list of topics._

### How to play ###

Sudoku is a puzzle game played on a 9x9 grid. The grid is evenly divided into 3x3 sub-blocks. Each space in the grid is filled with a digit from 1 to 9. No column, row, or block can contain the same digit more than once. Puzzles start with a number of digits already filled in and can often have more than one solution.

<p>
<table class="sudoku-puzzle">
  <tr>
    <td></td>
    <td></td>
    <td colspan="3"></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>3</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>2</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td>1</td>
    <td></td>
    <td>7</td>
    <td>9</td>
    <td></td>
    <td>4</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>2</td>
    <td>3</td>
    <td>9</td>
    <td></td>
    <td></td>
    <td>4</td>
    <td>8</td>
    <td></td>
    <td>7</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>1</td>
    <td></td>
    <td>2</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>9</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td>7</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>8</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>8</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>6</td>
    <td></td>
    <td>1</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>7</td>
    <td></td>
    <td></td>
    <td>2</td>
    <td></td>
    <td></td>
    <td>3</td>
    <td>1</td>
    <td>8</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
    <td>5</td>
    <td></td>
    <td>7</td>
    <td>8</td>
    <td></td>
    <td>6</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>9</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>6</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>
<em>An example Sudoku puzzle.</em>
</p>

When you play the game, a natural strategy is to jump around filling in digits where there can only be one answer. Easier puzzles can be solved with this method alone but harder puzzles force you to guess. Of course, a side effect of guessing means that sometimes you find out you're wrong and are forced to backtrack. A very tedious process. If you have an army of evil minions to delegate such tasks too, then uhh...just do that. For the rest of us, we'll have to settle for using a touch of math and code to much the same effect.

### Finding A Solution ###

Let's say you hung a puzzle on a wall. Then you threw a dart at the puzzle, hitting a completely random open space. Using a little set theory, you can determine all of the possible digits that could fit into that space.

<div class="review">
	<h4>Set Theory Refresher</h4>
	<p>Set theory is the study of sets. A set is an unordered collection of objects. Basic set theory is usually covered in the study of discrete mathematics.</p>
	<p>The union of two sets, denoted `uu`, is the set with all of the objects in each set. Consider the numbered sets `A` and `B`</p>
	<p>`A = {1,3,2}`</p>
	<p>`B = {5,4,3}`</p>
	<p>Then the union of these two sets is</p>
	<p>`A uu B = {1,2,3,4,5}`</p>
	<p>The universal set is a set containing all of the possible objects that should be considered and is denoted `U`. If, for example, only digits 1-9 should be considered, then</p>
	<p>`U = {1,2,3,4,5,6,7,8,9}`</p>
	<p>The absolute complement of a set is a set with all of the objects in the universal set but are NOT in another set. The absolute complement is written as `U \\\ A` or `A′` where `A` is an arbitrary set.</p>
	<p>`C = {2,4,6,8}`</p>
	<p>`C′ = U \\\ C = {1,2,3,4,5,6,7,8,9} \\\ {2,4,6,8} = {1,3,5,7,9}`</p>
</div>

Let the sets of digits in the current column, row, and block be denoted \`C\`, \`R\`, and \`B\` respectively. Then all of the unavailable digits are the union of these sets.

\`N = C uu R uu B\`

Since \`N\` is the set containing all of the unavailable digits, the absolute complement has to be all of the available digits.

\`A = N′\`

Here's an example using the puzzle from earlier. Assume the top-left corner is coordinate \`(1,1)\` and coordinates are ordered horizontally left-to-right and vertically top-to-bottom. Using the open space at \`(3,7)\`, start by finding the digits that are unavailable in the same column, row, and block.

<table class="sudoku-puzzle sudoku-puzzle-labeled">
  <tr>
    <td></td>
    <td></td>
    <td colspan="3"><div class="sudoku-puzzle-column">Column</div><div class="sudoku-puzzle-column-arrow"></div></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>3</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>2</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td>1</td>
    <td></td>
    <td>7</td>
    <td>9</td>
    <td></td>
    <td>4</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>2</td>
    <td>3</td>
    <td>9</td>
    <td></td>
    <td></td>
    <td>4</td>
    <td>8</td>
    <td></td>
    <td>7</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>1</td>
    <td></td>
    <td>2</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>9</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td>7</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>8</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>8</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>6</td>
    <td></td>
    <td>1</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>7</td>
    <td></td>
    <td></td>
    <td>2</td>
    <td></td>
    <td></td>
    <td>3</td>
    <td>1</td>
    <td>8</td>
    <td><span class="sudoku-puzzle-row">Row</span><div class="sudoku-puzzle-row-arrow"></div></td>
  </tr>
  <tr>
    <td><div class="sudoku-puzzle-block">Block<div class="sudoku-puzzle-block-arrow"></div></div></td>
    <td></td>
    <td></td>
    <td>5</td>
    <td></td>
    <td>7</td>
    <td>8</td>
    <td></td>
    <td>6</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>9</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>6</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

\`C = {9,2,5}\`

\`R = {7,2,3,1,8}\`

\`B = {7,5,9}\`

Union these sets together to find all unavailable digits.

\`N = C uu R uu B = {9,2,5} uu {7,2,3,1,8} uu {7,5,9} = {1,2,3,5,7,8,9}\`

Finally, compute the complement to get the available digits.

\`A = N′ = U \\\ N = {1,2,3,4,5,6,7,8,9} \\\ {1,2,3,5,7,8,9} = {4,6}\`

With a set of available digits in hand, one of the digits is removed from the set and put into the open space. Then the process is repeated at another open space. If no more spaces are open, the puzzle has been solved! If an open space has no available digits, one of the previous guesses must be wrong. Which means going back and trying a different digit. This process is called a [depth-first-search](https://en.wikipedia.org/wiki/Depth-first_search).

One question remains: does it matter how an open space is chosen? An optimal method would be to seek out the open spaces with a minimal number of available digits. Otherwise, many more paths have to be searched, as well as further, before being rejected and backtracking. But, the effort required to find such an open space might be greater than the time saved.

The simplest method, which works fine in practice, is to start at the top left and scan horizontally and then vertically. However, puzzles with many completely open blocks at the beginning will take a relatively long time to solve. Most puzzles are not like this but it can be a problem if the known digits are entered incorrectly (eg. character recognition fails because of poor lighting). In which case, a timeout might be necessary if a solution cannot be found within a reasonable amount of time.

### In Code Form ###

The state of the puzzle can be stored in a numbered 1D array of 9x9 elements where each value in the array represents its respective digit or zero if the space is empty. Using a 1D array to represent a 2D array of numbers is a pattern that will be very common later when working with images. A simple wrapper class makes querying the puzzle's state trivial. Since this class isn't used for an interactive game, there's no reason to check if each `Set()` operation keeps the game's state valid.

{% highlight c++ %}
#include <algorithm>
#include <array>

class Game
{
    public:
        static constexpr unsigned int WIDTH = 9;
        static constexpr unsigned int HEIGHT = 9;
        static constexpr unsigned int BLOCK_WIDTH = WIDTH / 3;
        static constexpr unsigned int BLOCK_HEIGHT = HEIGHT / 3;
        static constexpr unsigned char MAX_VALUE = 9;
        static constexpr unsigned char EMPTY_VALUE = 0;

        Game()
         : state()
        {
            Clear();
        }

        //Set all digits to EMPTY_VALUE.
        void Clear()
        {
            std::fill(state.begin(),state.end(),Game::EMPTY_VALUE);
        }

        //Set the space at (x,y) to a specific value.
        void Set(const unsigned int x,const unsigned int y,const unsigned char value)
        {
            if(x >= Game::WIDTH || y >= Game::HEIGHT || value > Game::MAX_VALUE)
                return;

            state[Index(x,y)] = value;
        }

        //Get the value at (x,y) or EMPTY_VALUE if blank.
        unsigned char Get(const unsigned int x,const unsigned int y) const
        {
            if(x >= Game::WIDTH || y >= Game::HEIGHT)
                return Game::EMPTY_VALUE;

            return state[Index(x,y)];
        }
    private:
        std::array<unsigned char,WIDTH * HEIGHT> state;

        static unsigned int Index(const unsigned int x,const unsigned int y)
        {
            return y * Game::WIDTH + x;
        }
};
{% endhighlight %}

The STL has a [std::set](http://en.cppreference.com/w/cpp/container/set) container and [std::set_difference](http://en.cppreference.com/w/cpp/algorithm/set_difference) algorithm to fulfill the mathematical set needs. Unfortunately, they are too slow in this case[^1]. Since the universal set is very small, at just 9 digits, and known ahead of time, a custom container can perform much better. The following implementation performs all insert, contains, and complement operations in [O(1)](https://en.wikipedia.org/wiki/Time_complexity#Constant_time) time.

{% highlight c++ %}
#include <algorithm>
#include <array>
#include <cassert>

//Set container for the valid Sudoku digits 1-9.
class DigitSet
{
    public:
        static constexpr unsigned int MAXIMUM_SIZE = 10;

        //Convenience class for iterating through values using the foreach syntax. This
        //should be designed so the zero digit never appears in the set.
        class const_iterator
        {
            ...
        };

        DigitSet()
        {
            std::fill(digits.begin(),digits.end(),0);
        }

        const_iterator begin() const
        {
            ...
        }

        const_iterator end() const
        {
            ...
        }

        void insert(const unsigned char digit)
        {
            assert(digit < digits.size());
            digits[digit] = 1;
        }

        bool contains(const unsigned char digit) const
        {
            assert(digit < digits.size());

            //Zero digit is not a valid digit.
            if(digit == 0)
                return false;

            return digits[digit] == 1;
        }

        DigitSet complemented() const
        {
            DigitSet complementedDigitSet;
            for(unsigned int x = 0;x < digits.size();x++)
            {
                complementedDigitSet.digits[x] = digits[x] ^ 1;
            }

            return complementedDigitSet;
        }
    private:
        std::array<unsigned char,MAXIMUM_SIZE> digits;
};
{% endhighlight %}

Finding the available digits is just a matter of querying the puzzle's state for all of the unavailable digits and then calculating the complement.

{% highlight c++ %}
void UnavailableRowDigits(const Game& game,
                          const unsigned int y,
                          DigitSet& unavailableDigits)
{
    for(unsigned int x = 0;x < Game::WIDTH;x++)
    {
        unavailableDigits.insert(game.Get(x,y));
    }
}

void UnavailableColumnDigits(const Game& game,
                             const unsigned int x,
                             DigitSet& unavailableDigits)
{
    for(unsigned int y = 0;y < Game::HEIGHT;y++)
    {
        unavailableDigits.insert(game.Get(x,y));
    }
}

void UnavailableBlockDigits(const Game& game,
                            const unsigned int x,
                            const unsigned int y,
                            DigitSet& unavailableDigits)
{
    const unsigned int blockStartX = (x / Game::BLOCK_WIDTH) * Game::BLOCK_WIDTH;
    const unsigned int blockStartY = (y / Game::BLOCK_HEIGHT) * Game::BLOCK_HEIGHT;
    for(unsigned int y = 0;y < Game::BLOCK_HEIGHT;y++)
    {
        for(unsigned int x = 0;x < Game::BLOCK_WIDTH;x++)
        {
            unavailableDigits.insert(game.Get(blockStartX + x,blockStartY + y));
        }
    }
}

DigitSet AvailableDigits(const Game& game,const unsigned int x,const unsigned int y)
{
    //Find the union of the known unavailable guesses.
    DigitSet unavailableDigits;
    UnavailableRowDigits(game,y,unavailableDigits);
    UnavailableColumnDigits(game,x,unavailableDigits);
    UnavailableBlockDigits(game,x,y,unavailableDigits);

    //Get the set of available guesses by computing the complement of the
    //unavailable guesses.
    return unavailableDigits.complemented();
}
{% endhighlight %}

Depth-first-search is easily performed using recursion. Start by selecting an open position. Find the available guesses. Then for each guess, apply it to the game's state and repeat the process recursively.

{% highlight C++ %}
#include <limits>

static bool SolveNext(Game& game,const unsigned int lastX,const unsigned int lastY)
{
    //Find the next position in the puzzle without a digit.
    unsigned int positionX = 0;
    unsigned int positionY = 0;
    if(!NextOpenPosition(game,lastX,lastY,positionX,positionY))
        return true; //No open positions, already solved?

    //Get the set of possible digits for this position.
    const DigitSet availableGuesses = AvailableGuesses(game,positionX,positionY);
    for(const unsigned char digit : availableGuesses)
    {
        //Try this digit.
        game.Set(positionX,positionY,digit);

        //Recursively keep searching at the next open position.
        if(SolveNext(game,positionX,positionY))
            return true;
    }

    //None of the attempted digits for this position were correct, clear it before
    //backtracking so it doesn't incorrectly influence other paths.
    game.Set(positionX,positionY,Game::EMPTY_VALUE);

    return false;
}

bool Solve(Game& game)
{
    //Use an overflow to start solving at (0,0).
    const unsigned int lastX = std::numeric_limits<unsigned int>::max();
    const unsigned int lastY = 0;

    //Recursively search for a solution in depth-first order.
    return SolveNext(game,lastX,lastY);
}
{% endhighlight %}

The next open space is determined by just scanning ahead in left-to-right and top-to-bottom order.

{% highlight C++ %}
static bool NextOpenPosition(const Game& game,
                             const unsigned int lastX,
                             const unsigned int lastY,
                             unsigned int& nextX,
                             unsigned int& nextY)
{
    for(unsigned int index = lastY * Game::WIDTH + lastX + 1;
        index < Game::WIDTH * Game::HEIGHT;
        index++)
    {
        nextX = index % Game::WIDTH;
        nextY = index / Game::HEIGHT;
        if(game.Get(nextX,nextY) == Game::EMPTY_VALUE)
            return true;
    }

    return false;
}
{% endhighlight %}

That's all there is to it! You might think with "Sudoku" in the title, this would be one of the biggest parts of the project. In fact, it's the smallest and easiest! Next time I'll start covering the augmented reality side by giving a crash course in image processing.

### Footnotes ###

[^1]: I originally used std::set instead of rolling my own but the performance was terrible. Waiting several seconds to find and show a solution is an unacceptable experience. Solving the "[World's Hardest Sudoku Problem](https://duckduckgo.com/?q=worlds+hardest+sudoku)" takes ~6 seconds with std::set and ~0 seconds using the method listed here. (Linux, GCC 6.3.1, Intel Core i5-3550)
