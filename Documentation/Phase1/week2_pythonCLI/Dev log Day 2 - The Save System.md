---
Date: 2026-01-07
Project: CLI Task Manager
Tech Stack: Python 3, JSON
Status: ✅ Persistence Online
Tags:
  - "[[JSON]]"
  - "[[File I/O]]"
  - "[[Serialization]]"
---
## 1. The Initiative
Today I built the **Memory System**.
Yesterday, the program had amnesia (data was lost on exit). Today, I implemented **File Persistence** using JSON.

## 2. The Mechanics (Serialization)
I learned two key concepts regarding data storage:
1.  **Serialization (`json.dump`):** Converting complex Python objects (Dictionaries) into a text format that can be stored on a hard drive.
2.  **Deserialization (`json.load`):** Reading that text back and reconstructing the Python object.

**The Code Implementation:**
```python
def save_data(self):
    with open(DATA_FILE, "w") as file:
        json.dump(self.task, file, indent=4)
````

## 3. The Logic Upgrade

I refactored the script into a `TaskManager` class.

- **Constructor (`__init__`):** Automatically loads data when the program starts.
    
- **Add Task:** Appends to the list AND immediately triggers a save.
    
- **List Tasks:** Reads from memory to display formatted output.
    

## 4. Visual Proof

The Code(taskmaster.py): ![Code](../../../Assets/Picture/week2/Main_code.png)

The Database (tasks.json): ![Data](../../../Assets/Picture/week2/sample_data.png)

The raw data persisting on the hard drive.

``` JSON
[
    {
        "id": 1,
        "title": "Buy Milk",
        "status": "pending",
        "completed": false,
        "created_at": "2026-01-06 13:17:36"
    }
]
```

