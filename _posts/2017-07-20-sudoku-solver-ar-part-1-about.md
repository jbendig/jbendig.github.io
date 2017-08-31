---
layout: post
title: "Sudoku Solver AR Part 1: About"
date: 2017-07-20 12:39:00 -0500
---

![Screenshot]({{ "/assets/sudokusolverar/screenshot.png" | relative_url }})

So you're working on a Sudoku problem and it dawns on you how repetitive the process is. You run out of obvious moves so you make a guess and continue on like that's the correct answer. A little later on you find out that you were wrong so you undo everything since your last guess, make another guess and continue on. Rinse and repeat until the puzzle is solved. Of course, as a programmer, you realize a program could do this much faster and it might even be more fun to write than doing the puzzle itself. Of course, having written said program, you've conquered Sudoku and will never touch another puzzle.

This is the start of a series of posts covering how to build an augmented reality Sudoku solver. The idea is to point a camera at a puzzle, have the computer find the puzzle, figure out which numbers are in which spots, solve the puzzle, and then display the answer over the original puzzle when displayed on a computer screen.

I'm going to be doing all of the processing from scratch. No computer vision libraries like OpenCV nor machine learning tools like TensorFlow will be used. This is an educational endeavor to cover how everything works. For serious projects, just use a battle tested library and save yourself the headaches.

Speaking of educational, there will be math. I plan to cover why things work and why sometimes they fall apart. You can scan past the parts you don't understand but I'll try to explain concepts or point you to places where you can develop a more detailed understanding.

All of the code is going to be in C++ and I'm going to assume basic knowledge of the STL. Using C++ means I can focus on writing clean code that's also fast enough. Tuning for maximum performance is a lot of fun but it's outside the scope of this project.

The source code for a complete working example can found on the [Sudoku Solver AR](https://github.com/jbendig/Sudoku-Solver-AR) GitHub page. The code is written for Linux and Windows but only the camera capture, window display, and some basic OpenGL parts are platform specific.

1. [About]({% post_url 2017-07-20-sudoku-solver-ar-part-1-about %})
2. [Sudoku]({% post_url 2017-07-27-sudoku-solver-ar-part-2-sudoku %})
3. [A Crash Course in Image Processing]({% post_url 2017-08-03-sudoku-solver-ar-part-3-a-crash-course-in-image-processing %})
4. [Camera Capture (Linux)]({% post_url 2017-08-10-sudoku-solver-ar-part-4-camera-capture-linux %})
5. [Camera Capture (Windows)]({% post_url 2017-08-17-sudoku-solver-ar-part-5-camera-capture-windows %})
6. [Canny Part 1 (Gaussian Blur)]({% post_url 2017-08-31-sudoku-solver-ar-part-6-canny-part-1-gaussian-blur %})
7. Canny Part 2 (Sobel)
8. Canny Part 3 (Non-Maximum Suppression)
9. Canny Part 4 (Connectivity Analysis)
10. Hough Transform
11. Finding Lines
12. Finding Puzzles
13. Extracting the Puzzle
14. Extracting the Digits
15. Rendering Solutions
16. Building Test Data
17. OCR with Nueral Networks
18. Compositing the Solution
