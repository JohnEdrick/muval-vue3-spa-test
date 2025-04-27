<template>
    <div class="p-6">
        <div class="flex space-x-6">
            <!-- LOOP through each STATUS -->
            <div v-for="(tasks, status) in tasksByStatus" :key="status" class="bg-gray-100 rounded-lg p-4 w-1/3"
                @dragover.prevent @drop="onDrop($event, status)">
                <h2 class="text-xl font-bold mb-4">{{ status }}</h2>

                <!-- LOOP through TASKS inside each STATUS -->
                <div v-for="task in tasks" :key="task.id" class="bg-white p-3 rounded shadow mb-2 cursor-move"
                    draggable="true" @dragstart="onDragStart($event, task)">

                    <h3 class="font-semibold">{{ task.title }}</h3>
                    <p v-if="task.description" class="text-sm text-gray-600">{{ task.description }}</p>

                    <div class="flex space-x-2 mt-2">
                        <button class="text-blue-500 text-xs" @click="editTask(task)">Edit</button>
                        <button class="text-red-500 text-xs" @click="deleteTask(task.id)">Delete</button>
                    </div>
                </div>
            </div>
        </div>

        <!-- Create / Edit Modal -->
        <div v-if="showModal" class="fixed inset-0 bg-black bg-opacity-40 flex items-center justify-center">
            <div class="bg-white p-6 rounded-lg w-96">
                <h2 class="text-lg font-bold mb-4">{{ editingTask.id ? 'Edit' : 'Create' }} Task</h2>

                <input v-model="editingTask.title" placeholder="Task Name" class="w-full p-2 border rounded mb-2" />
                <div v-if="errorMessages?.title" class="text-red-500 text-sm mb-2">{{ errorMessages.title[0] }}</div>

                <textarea v-model="editingTask.description" placeholder="Description (optional)"
                    class="w-full p-2 border rounded mb-2"></textarea>
                <div v-if="errorMessages?.description" class="text-red-500 text-sm mb-2">{{ errorMessages.description[0]
                }}</div>

                <select v-model="editingTask.status" class="w-full p-2 border rounded mb-2">
                    <option disabled value="">Select Status</option>
                    <option v-for="status in statuses" :key="status" :value="status">{{ status }}</option>
                </select>
                <div v-if="errorMessages?.status" class="text-red-500 text-sm mb-2">{{ errorMessages.status[0] }}</div>

                <select v-model="editingTask.user_id" class="w-full p-2 border rounded mb-4">
                    <option disabled value="">Select User</option>
                    <option v-for="user in users" :key="user.id" :value="user.id">{{ user.name }}</option>
                </select>
                <div v-if="errorMessages?.user_id" class="text-red-500 text-sm mb-2">{{ errorMessages.user_id[0] }}
                </div>

                <div class="flex justify-end space-x-2">
                    <button @click="showModal = false" class="px-4 py-2 bg-gray-300 rounded">Cancel</button>
                    <button @click="saveTask" class="px-4 py-2 bg-blue-500 text-white rounded">Save</button>
                </div>
            </div>
        </div>

        <!-- Add Task Button -->
        <button @click="openCreateModal"
            class="fixed bottom-6 right-6 bg-blue-500 text-white px-6 py-3 rounded-full shadow-lg hover:bg-blue-600">
            + Add Task
        </button>
    </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue';
import axios from 'axios';

// STATES
const tasks = ref([]);
const users = ref([]);
const showModal = ref(false);
const editingTask = reactive({ id: null, title: '', description: '', status: '', user_id: '' });
const draggedTask = ref(null);
const errorMessages = ref({});

// CONSTANTS
const statuses = ['To do', 'In Progress', 'Done'];
const API_BASE = 'http://127.0.0.1:8000/api/v1';
const TASKS_API = `${API_BASE}/tasks`;
const USERS_API = `${API_BASE}/users`;

// COMPUTED
const tasksByStatus = computed(() => {
    if (!tasks.value.length) return {};
    const grouped = {};
    statuses.forEach(status => {
        grouped[status] = tasks.value.filter(task => task.status === status);
    });
    return grouped;
});

// API FUNCTIONS
async function fetchTasks() {
    try {
        const response = await axios.get(TASKS_API);
        tasks.value = response.data.tasks;
    } catch (error) {
        console.error('Error fetching tasks:', error);
    }
}

async function fetchUsers() {
    try {
        const response = await axios.get(USERS_API);
        users.value = response.data;
    } catch (error) {
        console.error('Error fetching users:', error);
    }
}

async function saveTask() {
    try {
        errorMessages.value = {}; // Clear previous errors
        if (editingTask.id) {
            await axios.put(`${TASKS_API}/${editingTask.id}`, editingTask);
            const idx = tasks.value.findIndex(t => t.id === editingTask.id);
            if (idx !== -1) tasks.value[idx] = { ...editingTask };
        } else {
            const response = await axios.post(TASKS_API, editingTask);
            tasks.value.push(response.data);
        }
        fetchTasks();
        showModal.value = false;
    } catch (error) {
        if (error.response && error.response.data.errors) {
            errorMessages.value = error.response.data.errors;
        } else {
            console.error('Error saving task:', error);
        }
    }
}

async function deleteTask(taskId) {
    if (!confirm('Are you sure you want to delete this task?')) return;
    try {
        await axios.delete(`${TASKS_API}/${taskId}`);
        tasks.value = tasks.value.filter(task => task.id !== taskId);
    } catch (error) {
        console.error('Error deleting task:', error);
    }
}

// DRAG AND DROP FUNCTIONS
function onDragStart(event, task) {
    draggedTask.value = task;
    event.dataTransfer.effectAllowed = "move";
    event.dataTransfer.setData("text/plain", task.id);
}

async function onDrop(event, status) {
    event.preventDefault();
    const taskId = parseInt(event.dataTransfer.getData("text/plain"), 10);
    const task = tasks.value.find(t => t.id === taskId);

    if (task) {
        try {
            task.status = status;
            await axios.put(`${TASKS_API}/${task.id}`, task);
        } catch (error) {
            console.error('Error updating task status:', error);
        }
    }
    draggedTask.value = null;
}

// UTILS
function getUser(userId) {
    return users.value.find(user => user.id === userId);
}

function openCreateModal() {
    Object.assign(editingTask, { id: null, title: '', description: '', status: '', user_id: '' });
    errorMessages.value = {};
    showModal.value = true;
}

function editTask(task) {
    Object.assign(editingTask, task);
    errorMessages.value = {};
    showModal.value = true;
}

// LIFECYCLE
onMounted(() => {
    fetchUsers();
    fetchTasks();
});
</script>
