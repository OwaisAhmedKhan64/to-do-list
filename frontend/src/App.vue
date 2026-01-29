<script setup>
import { ref, onMounted } from 'vue';
import axios from 'axios';

const API_URL = 'http://127.0.0.1:8000/tasks/';
const todos = ref([]);

// --- 1. ADD TASK LOGIC ---
const newTaskTitle = ref(''); // Stores the text from the "Add" input box

const addTask = async () => {
  // Prevent adding empty tasks
  if (newTaskTitle.value.trim() === '') return;

  try {
    // Send POST request to create new task
    const response = await axios.post(API_URL, {
      title: newTaskTitle.value,
      is_completed: false
    });

    // Add the new task (with its new ID from backend) to our local list
    todos.value.push(response.data);
    
    // Clear the input box
    newTaskTitle.value = '';
  } catch (error) {
    console.error('Error adding task:', error);
  }
};

// --- 2. DELETE TASK LOGIC ---
const deleteTask = async (id) => {
  if (!confirm('Are you sure you want to delete this task?')) return;

  try {
    // Send DELETE request to specific ID
    await axios.delete(`${API_URL}${id}/`);
    
    // Remove the item from our local list immediately
    todos.value = todos.value.filter(todo => todo.id !== id);
  } catch (error) {
    console.error('Error deleting task:', error);
  }
};

// --- 3. EDIT TASK LOGIC ---
const editingTaskId = ref(null); // Tracks WHICH task is currently being edited
const editingTitle = ref('');    // Stores the temporary title while editing

// Turn on "Edit Mode" for a specific task
const startEditing = (todo) => {
  editingTaskId.value = todo.id;
  editingTitle.value = todo.title; // Copy existing title to input
};

// Save the changes
const saveEdit = async (todo) => {
  try {
    // Send PUT request with updated title
    const response = await axios.put(`${API_URL}${todo.id}/`, {
      title: editingTitle.value,
      description: todo.description, // We must keep existing fields
      is_completed: todo.is_completed
    });

    // Update the local list with the new data from backend
    // This finds the index of the task and replaces it
    const index = todos.value.findIndex(t => t.id === todo.id);
    if (index !== -1) {
      todos.value[index] = response.data;
    }

    cancelEdit(); // Turn off edit mode
  } catch (error) {
    console.error('Error updating task:', error);
  }
};

// Cancel editing without saving
const cancelEdit = () => {
  editingTaskId.value = null;
  editingTitle.value = '';
};

// --- 4. TOGGLE STATUS LOGIC (Existing) ---
const toggleTodoStatus = async (todo) => {
  const originalStatus = todo.is_completed;
  todo.is_completed = !todo.is_completed; // Optimistic update

  try {
    await axios.put(`${API_URL}${todo.id}/`, todo);
  } catch (error) {
    todo.is_completed = originalStatus; // Revert on error
    console.error('Error toggling status:', error);
  }
};

// Fetch initial data
onMounted(async () => {
  try {
    const response = await axios.get(API_URL);
    todos.value = response.data;
  } catch (error) {
    console.error('Error fetching tasks:', error);
  }
});
</script>

<template>
  <main>
    <h1>My To-Do List</h1>

    <!-- ADD TASK SECTION -->
    <div class="add-task-container">
      <input 
        v-model="newTaskTitle" 
        @keyup.enter="addTask"
        placeholder="What needs to be done?" 
        class="add-input"
      />
      <button @click="addTask" class="add-btn">Add</button>
    </div>
    
    <!-- TASK LIST -->
    <ul class="todo-list">
      <li v-for="todo in todos" :key="todo.id" :class="{ completed: todo.is_completed }">
        
        <!-- CHECKBOX (Always visible) -->
        <input 
          type="checkbox" 
          :checked="todo.is_completed" 
          @change="toggleTodoStatus(todo)"
        />

        <!-- EDIT MODE: Shows Input field if this specific task is being edited -->
        <div v-if="editingTaskId === todo.id" class="edit-mode">
          <input 
            v-model="editingTitle" 
            @keyup.enter="saveEdit(todo)"
            class="edit-input"
          />
          <button @click="saveEdit(todo)" class="save-btn">💾</button>
          <button @click="cancelEdit" class="cancel-btn">❌</button>
        </div>

        <!-- VIEW MODE: Shows Text + Buttons if NOT being edited -->
        <div v-else class="view-mode">
          <span class="todo-title" @dblclick="startEditing(todo)">
            {{ todo.title }}
          </span>
          <div class="actions">
            <button @click="startEditing(todo)" class="edit-btn">✏️</button>
            <button @click="deleteTask(todo.id)" class="delete-btn">🗑️</button>
          </div>
        </div>

      </li>
    </ul>
  </main>
</template>

<style scoped>
main {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  max-width: 600px;
  margin: 3rem auto;
  background: #fff;
  padding: 2rem;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}

h1 { text-align: center; color: #333; }

/* Add Task Styles */
.add-task-container { display: flex; gap: 10px; margin-bottom: 2rem; }
.add-input { flex-grow: 1; padding: 10px; border: 1px solid #ddd; border-radius: 4px; }
.add-btn { padding: 10px 20px; background-color: #42b983; color: white; border: none; border-radius: 4px; cursor: pointer; }
.add-btn:hover { background-color: #3aa876; }

/* List Styles */
.todo-list { list-style: none; padding: 0; }
li { display: flex; align-items: center; padding: 1rem 0; border-bottom: 1px solid #eee; }
input[type="checkbox"] { transform: scale(1.5); margin-right: 15px; cursor: pointer;}

/* Modes */
.view-mode, .edit-mode { display: flex; align-items: center; flex-grow: 1; justify-content: space-between; }
.todo-title { font-size: 1.1rem; cursor: pointer; }
.completed .todo-title { text-decoration: line-through; color: #aaa; }

/* Buttons */
.actions button, .edit-mode button { margin-left: 5px; background: none; border: none; font-size: 1.2rem; cursor: pointer; }
.actions button:hover { transform: scale(1.2); }
.delete-btn:hover { color: red; }
.edit-input { padding: 5px; font-size: 1rem; flex-grow: 1; margin-right: 10px;}
</style>