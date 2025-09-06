Understood! Here is your text, rewritten for greater fluency, while strictly preserving your instructions:

---

## Implementation

- Your code MUST prioritize readability and maintainability.
- Your code MUST be well-documented.
- Your code MUST be well-commented.
- The [spec](./docs/spec.md) is authoritative.
- Before you start writing code, you MUST first create a plan of [TODOs](./docs/todos.md), and keep it updated as you work, making changes whenever necessary.

## Testing

- Your tests MUST be thorough and cover every scenario in the acceptance test cases.
- Your unit tests MUST test only one thing at a time.
- Your E2E tests may cover multiple aspects or workflows in a single scenario.
- Your E2E tests are designed to validate the CLI.
- You MAY use a Docker container for your E2E tests to manage the environment, but your unit tests must use mocks when necessary.

## Comments

- Use comments to label sections of code; do not add comments to every line.
- Do NOT use comments that simply restate what the code is already doing—you MUST depend on readability and short, descriptive names instead.
- Split logical sections of code with "---" comments that start with five dashes and are exactly 80 characters long.
- Add concise, descriptive comments to GROUPS of code in the format "(imperative verb) ...(concise description)...because (reason)". Keep these SHORT and concise.

### Examples of Bad Comments

```python
# This is a Python program. Python is a programming language.

# Import the 'random' module so we can use random numbers.
import random  # 'random' is now ready to use.

# Define a function called 'add' which takes two arguments: a and b.
def add(a, b):
    # Add a and b together using the + operator.
    result = a + b  # This computes the sum.
    # Return the result back to whoever called this function.
    return result  # Give the answer back.

# Define a list of numbers from 1 to 5.
numbers = [1, 2, 3, 4, 5]  # This is a list. Lists can store things.

# Pick a random number from our list.
chosen_number = random.choice(numbers)  # The computer chooses a number.

# Print out what the chosen number is. Printing makes stuff appear on the screen.
print("The chosen number is:", chosen_number)  # Show the number to the human.

# Add 10 to the chosen number using our add function.
final_result = add(chosen_number, 10)  # Add 10 for no reason.

# Print out the final result. Again, this will be shown to the human.
print("The final result after adding 10 is:", final_result)  # More output!

# End of program. There is no more code down here.
```

### Examples of Good Comments

```python
import random

# ----- FUNCTION DEFINITIONS --------------------------------------------------
def add(x, y):
    return x + y

def pick_random(num_list):
    return random.choice(num_list)

def multiply(x, factor):
    return x * factor

# ----- MAIN LOGIC ------------------------------------------------------------
# Generate, pick, and process numbers because we want random math
numbers = [n for n in range(1, 11) if n % 2 == 0]
chosen = pick_random(numbers)
summed = add(chosen, 10)
product = multiply(summed, random.randint(1, 5))

# Print results because we need output
print(f"Chosen: {chosen}")
print(f"Summed: {summed}")
print(f"Product: {product}")
```
