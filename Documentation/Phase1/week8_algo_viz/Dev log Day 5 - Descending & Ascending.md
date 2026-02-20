---
date: 2026-02-20
project: Algorithm Visualizer
topic: Directional Sorting & Final Polish
Tags:
  - "[[Python]]"
  - "[[Algorithms]]"
  - "[[Logic]]"
  - "[[State Management]]"
  - "[[Dev Log]]"
  - "[[Documentation]]"
---
# 📝 DEV LOG: WEEK 08 - DAY 5

**Focus:** Implementing Ascending/Descending directional logic and finalizing the application.

## 1. The Initiative
To finish the tool, I needed the ability to sort backwards. Instead of creating entirely new functions for reverse sorting, the goal was to make the existing algorithms smart enough to accept a "direction" flag and flip their comparison logic dynamically.

## 2. The Concepts

### Concept A: The Magic Flip (Conditional Logic)
Sorting boils down to comparing two numbers. To change the direction of the sort, I only needed to reverse the inequality operator.
* **Ascending:** `(ascending and num1 > num2)`
* **Descending:** `(not ascending and num1 < num2)`
By combining these with an `or` statement, the algorithm seamlessly handles both directions without duplicate loops.

### Concept B: Passing State to Generators
A bug I encountered taught me how generators hold state. When pressing the spacebar, I had to pass the `self.ascending` variable directly into the algorithm function `self.sorting_algorithm(self.draw_info, self.ascending)`. If I didn't, the generator would snapshot the default state and ignore any key presses I made beforehand.

## 3. The Output
The completed **Sorting Algorithm Visualizer (v1.0)**.
* **Scale:** Dynamically scales up to 100+ items.
* **Algorithms:** Bubble, Insertion, and Selection Sort.
* **Controls:** Full keyboard mapping for resets, start/stop, algorithm switching, and bi-directional sorting.
* **UI:** Real-time HUD and repeating bar gradients for visual clarity.

![Day5.1](../../../Picture/week8/day5.1.png)

![Day5.2](../../../Picture/week8/day5.2.png)


---

## 4. Source Code (Final Version)

```python
import pygame
import math
import random
import sys
from pygame.locals import * # type: ignore

# --- GLOBAL CONFIGURATION ---
BLACK = (0, 0, 0)
WHITE = (255, 255, 255)
GREEN = (0, 255, 0)
RED = (255, 0, 0)
BLUE = (0, 0, 255)

class DrawInformation:
    SIDE_PAD = 100
    TOP_PAD = 150
    
    GRADIENTS = [
        (128, 128, 128),
        (160, 160, 160),
        (192, 192, 192)
    ]

    def __init__(self, width, height, lst):
        self.width = width
        self.height = height
        
        self.window = pygame.display.get_surface()
        self.set_list(lst)
        
        self.font = pygame.font.SysFont('Ariel', 30)
        self.large_font = pygame.font.SysFont('Ariel', 40)

    def set_list(self, lst):
        self.lst = lst
        self.min_val = min(lst)
        self.max_val = max(lst)

        self.block_width = round((self.width - self.SIDE_PAD) / len(lst))
        self.block_height = math.floor((self.height - self.TOP_PAD) / (self.max_val - self.min_val))
        self.start_x = self.SIDE_PAD // 2

    def draw(self, bg_color, algo_name, ascending, color_positions={}):
        self.window.fill(bg_color)
        
        direction = "Ascending" if ascending else "Descending"
        title_text = f"{algo_name} - {direction}"
        controls_1 = "R: Reset | SPACE: Start | A: Ascending | D: Descending"
        controls_2 = "I: Insertion | B: Bubble | S: Selection"
        
        title_surface = self.large_font.render(title_text, 1, BLUE)
        controls_1_surface = self.font.render(controls_1, 1, WHITE)
        controls_2_surface = self.font.render(controls_2, 1, WHITE)
        
        self.window.blit(title_surface, (self.width/2 - title_surface.get_width()/2, 5))
        self.window.blit(controls_1_surface, (self.width/2 - controls_1_surface.get_width()/2, 45))
        self.window.blit(controls_2_surface, (self.width/2 - controls_2_surface.get_width()/2, 75))
        
        for i, val in enumerate(self.lst):
            x = self.start_x + i * self.block_width
            y = self.height - (val - self.min_val) * self.block_height
            
            color = self.GRADIENTS[i % 3]
            
            if i in color_positions:
                color = color_positions[i]
                
            pygame.draw.rect(self.window, color, (x, y, self.block_width, self.height))
            
        pygame.display.update()

# --- ALGORITHMS ---
def bubble_sort(draw_info, ascending=True):
    lst = draw_info.lst
    for i in range(len(lst) - 1):
        for j in range(len(lst) - 1 - i):
            num1 = lst[j]
            num2 = lst[j + 1]
            
            if (ascending and num1 > num2) or (not ascending and num1 < num2):
                lst[j], lst[j + 1] = lst[j + 1], lst[j]
                draw_info.draw(BLACK, "Bubble Sort", ascending, {j: GREEN, j+1: RED})
                yield True
    return lst

def insertion_sort(draw_info, ascending=True):
    lst = draw_info.lst
    for i in range(1, len(lst)):
        current = lst[i]
        while True:
            if i == 0:
                break
            
            if ascending and lst[i - 1] > lst[i]:
                swap = True
            elif not ascending and lst[i - 1] < lst[i]:
                swap = True
            else:
                swap = False
                
            if not swap:
                break
            
            lst[i], lst[i - 1] = lst[i - 1], lst[i]
            draw_info.draw(BLACK, "Insertion Sort", ascending, {i - 1: GREEN, i: RED})
            yield True
            i -= 1
    return lst

def selection_sort(draw_info, ascending=True):
    lst = draw_info.lst
    for i in range(len(lst) - 1):
        target_idx = i
        for j in range(i + 1, len(lst)):
            if (ascending and lst[j] < lst[target_idx]) or (not ascending and lst[j] > lst[target_idx]):
                target_idx = j
                
        if target_idx != i:
            lst[i], lst[target_idx] = lst[target_idx], lst[i]
            draw_info.draw(BLACK, "Selection Sort", ascending, {i: GREEN, target_idx: RED})
            yield True
    return lst

class Main:
    pygame.init()
    
    BACKGROUND_COLOR = BLACK
    DISPLAY_WIDTH = 800
    DISPLAY_HEIGHT = 600
    DISPLAY = pygame.display.set_mode((DISPLAY_WIDTH, DISPLAY_HEIGHT))
    
    def __init__(self):
        pygame.display.set_caption("Sorting Algorithm Visualizer")
        self.lst = self.generate_starting_list()
        self.draw_info = DrawInformation(self.DISPLAY_WIDTH, self.DISPLAY_HEIGHT, self.lst)
        
        self.sorting = False
        self.ascending = True 
        self.sorting_algorithm = bubble_sort
        self.sorting_algo_name = "Bubble Sort"
        self.sorting_algorithm_generator = None
        
    def generate_starting_list(self):
        return [random.randint(0, 200) for _ in range(100)]

    def run(self):
        clock = pygame.time.Clock()
        
        while True:
            clock.tick(60)
            
            if self.sorting:
                try:
                    next(self.sorting_algorithm_generator) # type: ignore
                except StopIteration:
                    self.sorting = False
            else:
                self.draw_info.draw(BLACK, self.sorting_algo_name, self.ascending)
            
            for event in pygame.event.get():
                if event.type == QUIT: #type: ignore
                    pygame.quit()
                    sys.exit()
                
                if event.type == KEYDOWN: #type: ignore
                    if event.key == K_r: #type: ignore
                        self.lst = self.generate_starting_list()
                        self.draw_info.set_list(self.lst)
                        self.sorting = False
                        
                    if event.key == K_SPACE and not self.sorting: #type: ignore
                        self.sorting = True
                        self.sorting_algorithm_generator = self.sorting_algorithm(self.draw_info, self.ascending)
                        
                    elif event.key == K_a and not self.sorting: #type: ignore
                        self.ascending = True
                        
                    elif event.key == K_d and not self.sorting: #type: ignore
                        self.ascending = False
                        
                    elif event.key == K_i and not self.sorting: #type: ignore
                        self.sorting_algorithm = insertion_sort
                        self.sorting_algo_name = "Insertion Sort"
                    
                    elif event.key == K_b and not self.sorting: #type: ignore
                        self.sorting_algorithm = bubble_sort
                        self.sorting_algo_name = "Bubble Sort"
                        
                    elif event.key == K_s and not self.sorting: #type: ignore
                        self.sorting_algorithm = selection_sort
                        self.sorting_algo_name = "Selection Sort"

if __name__ == "__main__":
    app = Main()
    app.run()
````

