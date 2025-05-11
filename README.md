<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Kanban Task Tracker</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6; /* Tailwind gray-100 */
        }
        .kanban-column {
            min-height: 300px; /* Ensure columns have a minimum height */
            background-color: #e5e7eb; /* Tailwind gray-200 */
            border-radius: 0.5rem; /* Tailwind rounded-lg */
            padding: 1rem; /* Tailwind p-4 */
            transition: background-color 0.2s ease-in-out;
        }
        .kanban-column.drag-over {
            background-color: #d1d5db; /* Tailwind gray-300 */
        }
        .task-card {
            background-color: white;
            border-radius: 0.375rem; /* Tailwind rounded-md */
            padding: 0.75rem; /* Tailwind p-3 */
            margin-bottom: 0.75rem; /* Tailwind mb-3 */
            box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1), 0 1px 2px 0 rgba(0, 0, 0, 0.06); /* Tailwind shadow */
            cursor: grab;
            transition: transform 0.2s ease-in-out, box-shadow 0.2s ease-in-out;
        }
        .task-card:active {
            cursor: grabbing;
            transform: scale(1.05);
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05); /* Tailwind shadow-lg */
        }
        .task-card.dragging {
            opacity: 0.5;
        }
        .task-placeholder {
            background-color: rgba(0,0,0,0.1);
            border: 2px dashed #9ca3af; /* Tailwind gray-400 */
            border-radius: 0.375rem;
            height: 50px;
            margin-bottom: 0.75rem;
        }

        #confetti-canvas {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none; /* Allows clicks to go through the canvas */
            z-index: 9999;
            display: none; /* Hidden by default */
        }
        .delete-task-btn {
            background: none;
            border: none;
            color: #ef4444; /* Tailwind red-500 */
            cursor: pointer;
            font-size: 0.875rem; /* Tailwind text-sm */
            padding: 0.25rem; /* Tailwind p-1 */
            float: right;
            opacity: 0.7;
            transition: opacity 0.2s;
        }
        .delete-task-btn:hover {
            opacity: 1;
        }
    </style>
</head>
<body class="min-h-screen p-4 sm:p-6 lg:p-8">
    <div class="container mx-auto max-w-6xl">
        <header class="mb-8 text-center">
            <h1 class="text-4xl font-bold text-gray-800">Kanban Task Tracker</h1>
        </header>

        <div class="mb-8 p-6 bg-white rounded-lg shadow-md">
            <h2 class="text-2xl font-semibold text-gray-700 mb-4">Add New Task</h2>
            <div class="flex flex-col sm:flex-row gap-4">
                <input type="text" id="new-task-input" placeholder="Enter task description..." class="flex-grow p-3 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none" aria-label="New task description">
                <button id="add-task-btn" class="bg-blue-500 hover:bg-blue-600 text-white font-semibold py-3 px-6 rounded-md transition duration-150 ease-in-out">
                    Add Task
                </button>
            </div>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-3 gap-6" id="kanban-board">
            <div id="todo-column" class="kanban-column" data-column-id="todo">
                <h2 class="text-xl font-semibold text-gray-700 mb-4 border-b-2 border-red-500 pb-2">To Do</h2>
                <div class="task-list space-y-3" data-column-id="todo-tasks">
                    </div>
            </div>

            <div id="inprogress-column" class="kanban-column" data-column-id="inprogress">
                <h2 class="text-xl font-semibold text-gray-700 mb-4 border-b-2 border-yellow-500 pb-2">In Progress</h2>
                <div class="task-list space-y-3" data-column-id="inprogress-tasks">
                    </div>
            </div>

            <div id="done-column" class="kanban-column" data-column-id="done">
                <h2 class="text-xl font-semibold text-gray-700 mb-4 border-b-2 border-green-500 pb-2">Done</h2>
                <div class="task-list space-y-3" data-column-id="done-tasks">
                    </div>
            </div>
        </div>
    </div>

    <canvas id="confetti-canvas"></canvas>

    <script>
        // --- DOM Elements ---
        const newTaskInput = document.getElementById('new-task-input');
        const addTaskBtn = document.getElementById('add-task-btn');
        const kanbanBoard = document.getElementById('kanban-board');
        const todoTasksList = document.querySelector('[data-column-id="todo-tasks"]');
        const inprogressTasksList = document.querySelector('[data-column-id="inprogress-tasks"]');
        const doneTasksList = document.querySelector('[data-column-id="done-tasks"]');
        const columns = document.querySelectorAll('.kanban-column .task-list');
        const confettiCanvas = document.getElementById('confetti-canvas');
        const confettiCtx = confettiCanvas.getContext('2d');

        // --- State ---
        let tasks = []; // Array to store tasks {id: string, text: string, column: string}
        let draggedTask = null; // Stores the task element being dragged
        let placeholder = null; // Placeholder element for drag indication

        // --- Task Functions ---
        function generateId() {
            return 'task-' + Math.random().toString(36).substr(2, 9) + Date.now();
        }

        function createTaskElement(task) {
            const taskCard = document.createElement('div');
            taskCard.classList.add('task-card');
            taskCard.setAttribute('draggable', 'true');
            taskCard.setAttribute('id', task.id);
            taskCard.dataset.taskId = task.id; // Store task id for easier access

            const taskText = document.createElement('p');
            taskText.textContent = task.text;
            taskText.classList.add('text-gray-800', 'break-words'); // Allow long words to break

            const deleteBtn = document.createElement('button');
            deleteBtn.innerHTML = '&times;'; // Multiplication sign for 'x'
            deleteBtn.classList.add('delete-task-btn');
            deleteBtn.setAttribute('aria-label', 'Delete task');
            deleteBtn.onclick = (event) => {
                event.stopPropagation(); // Prevent drag start
                deleteTask(task.id);
            };
            
            taskCard.appendChild(deleteBtn);
            taskCard.appendChild(taskText);

            // Add drag event listeners
            taskCard.addEventListener('dragstart', handleDragStart);
            taskCard.addEventListener('dragend', handleDragEnd);
            
            // Touch event listeners for mobile drag & drop
            taskCard.addEventListener('touchstart', handleTouchStart, { passive: false });


            return taskCard;
        }
        
        function addTask() {
            const taskText = newTaskInput.value.trim();
            if (taskText === '') {
                // Simple feedback if input is empty - can be enhanced with a styled message
                newTaskInput.classList.add('border-red-500');
                setTimeout(() => newTaskInput.classList.remove('border-red-500'), 2000);
                return;
            }

            const newTask = {
                id: generateId(),
                text: taskText,
                column: 'todo' // Default column
            };

            tasks.push(newTask);
            renderTasks();
            saveTasksToLocalStorage();
            newTaskInput.value = ''; // Clear input
            newTaskInput.focus();
        }

        function deleteTask(taskId) {
            tasks = tasks.filter(task => task.id !== taskId);
            renderTasks();
            saveTasksToLocalStorage();
        }

        function updateTaskColumn(taskId, newColumnId) {
            const task = tasks.find(t => t.id === taskId);
            if (task) {
                task.column = newColumnId;
                if (newColumnId === 'done') {
                    triggerConfetti();
                }
            }
            renderTasks(); // Re-render to reflect changes, could be optimized
            saveTasksToLocalStorage();
        }


        // --- Render Functions ---
        function renderTasks() {
            // Clear existing tasks from lists
            todoTasksList.innerHTML = '';
            inprogressTasksList.innerHTML = '';
            doneTasksList.innerHTML = '';

            tasks.forEach(task => {
                const taskElement = createTaskElement(task);
                if (task.column === 'todo') {
                    todoTasksList.appendChild(taskElement);
                } else if (task.column === 'inprogress') {
                    inprogressTasksList.appendChild(taskElement);
                } else if (task.column === 'done') {
                    doneTasksList.appendChild(taskElement);
                }
            });
        }

        // --- Drag and Drop Handlers (HTML5 API) ---
        function handleDragStart(event) {
            draggedTask = event.target;
            event.dataTransfer.setData('text/plain', event.target.id);
            event.dataTransfer.effectAllowed = 'move';
            setTimeout(() => { // Make the original element less visible
                draggedTask.classList.add('dragging');
            }, 0);

            // Create placeholder
            placeholder = document.createElement('div');
            placeholder.classList.add('task-placeholder');
            placeholder.style.height = `${draggedTask.offsetHeight}px`;
        }

        function handleDragEnd(event) {
            if (draggedTask) {
                draggedTask.classList.remove('dragging');
            }
            draggedTask = null;
            if (placeholder && placeholder.parentNode) {
                placeholder.parentNode.removeChild(placeholder);
            }
            placeholder = null;
            columns.forEach(column => column.closest('.kanban-column').classList.remove('drag-over'));
        }

        function handleDragOver(event) {
            event.preventDefault(); // Necessary to allow dropping
            event.dataTransfer.dropEffect = 'move';
            
            const columnElement = event.target.closest('.kanban-column');
            if (!columnElement) return;

            const taskList = columnElement.querySelector('.task-list');
            if (!taskList) return;

            columns.forEach(col => col.closest('.kanban-column').classList.remove('drag-over'));
            columnElement.classList.add('drag-over');

            // Placeholder logic
            if (placeholder) {
                const rect = event.target.getBoundingClientRect();
                const isTaskCard = event.target.classList.contains('task-card');
                const offsetY = event.clientY - rect.top;

                if (isTaskCard && event.target !== placeholder) {
                    if (offsetY > event.target.offsetHeight / 2) {
                        taskList.insertBefore(placeholder, event.target.nextSibling);
                    } else {
                        taskList.insertBefore(placeholder, event.target);
                    }
                } else if (!isTaskCard && taskList.contains(event.target) && taskList.children.length > 0) {
                    // If dragging over empty space in the list or the list itself
                    const childrenArray = Array.from(taskList.children).filter(child => child !== placeholder && child !== draggedTask);
                    let inserted = false;
                    for (let child of childrenArray) {
                        const childRect = child.getBoundingClientRect();
                        if (event.clientY < childRect.top + childRect.height / 2) {
                            taskList.insertBefore(placeholder, child);
                            inserted = true;
                            break;
                        }
                    }
                    if (!inserted) {
                        taskList.appendChild(placeholder);
                    }
                } else if (taskList.children.length === 0 || (taskList.children.length === 1 && taskList.firstChild === placeholder)) {
                     taskList.appendChild(placeholder); // If column is empty or only contains placeholder
                }
            }
        }

        function handleDrop(event) {
            event.preventDefault();
            const columnElement = event.target.closest('.kanban-column');
            if (!columnElement || !draggedTask) return;

            const targetTaskList = columnElement.querySelector('.task-list');
            const targetColumnId = columnElement.dataset.columnId;
            const taskId = draggedTask.dataset.taskId;

            if (targetTaskList && taskId) {
                if (placeholder && placeholder.parentNode) {
                     targetTaskList.insertBefore(draggedTask, placeholder); // Insert before placeholder
                } else {
                    targetTaskList.appendChild(draggedTask); // Fallback if no placeholder
                }
                updateTaskColumn(taskId, targetColumnId);
            }

            if (placeholder && placeholder.parentNode) {
                placeholder.parentNode.removeChild(placeholder);
            }
            placeholder = null;
            draggedTask.classList.remove('dragging');
            draggedTask = null;
            columns.forEach(col => col.closest('.kanban-column').classList.remove('drag-over'));
        }

        // --- Touch Drag and Drop Handlers ---
        let touchDragElement = null;
        let touchClone = null;
        let initialTouchX, initialTouchY;
        let offsetX, offsetY; // offset from top-left of element to touch point

        function handleTouchStart(event) {
            if (event.touches.length !== 1) return; // Only handle single touch
            event.preventDefault(); // Prevent scrolling while dragging

            touchDragElement = event.target.closest('.task-card');
            if (!touchDragElement) return;

            const rect = touchDragElement.getBoundingClientRect();
            initialTouchX = event.touches[0].clientX;
            initialTouchY = event.touches[0].clientY;
            offsetX = initialTouchX - rect.left;
            offsetY = initialTouchY - rect.top;

            touchClone = touchDragElement.cloneNode(true);
            touchClone.style.position = 'absolute';
            touchClone.style.zIndex = '1000';
            touchClone.style.opacity = '0.8';
            touchClone.style.pointerEvents = 'none'; // So it doesn't interfere with touchmove target
            touchClone.style.width = `${rect.width}px`;
            touchClone.style.height = `${rect.height}px`;
            document.body.appendChild(touchClone);
            moveClone(event.touches[0].clientX, event.touches[0].clientY);

            touchDragElement.classList.add('dragging'); // Make original less visible

            // Create placeholder for touch
            placeholder = document.createElement('div');
            placeholder.classList.add('task-placeholder');
            placeholder.style.height = `${touchDragElement.offsetHeight}px`;
            // Insert placeholder where the original element was
            touchDragElement.parentNode.insertBefore(placeholder, touchDragElement.nextSibling);


            document.addEventListener('touchmove', handleTouchMove, { passive: false });
            document.addEventListener('touchend', handleTouchEnd);
        }

        function moveClone(clientX, clientY) {
            if (touchClone) {
                touchClone.style.left = `${clientX - offsetX}px`;
                touchClone.style.top = `${clientY - offsetY}px`;
            }
        }

        function handleTouchMove(event) {
            if (!touchDragElement || !touchClone || event.touches.length !== 1) return;
            event.preventDefault();

            moveClone(event.touches[0].clientX, event.touches[0].clientY);

            // Determine target column based on touch position
            touchClone.style.display = 'none'; // Temporarily hide clone to get element underneath
            const elementUnderTouch = document.elementFromPoint(event.touches[0].clientX, event.touches[0].clientY);
            touchClone.style.display = ''; // Show clone again

            columns.forEach(colList => colList.closest('.kanban-column').classList.remove('drag-over'));
            
            if (elementUnderTouch) {
                const targetColumnElement = elementUnderTouch.closest('.kanban-column');
                if (targetColumnElement) {
                    targetColumnElement.classList.add('drag-over');
                    const targetTaskList = targetColumnElement.querySelector('.task-list');
                    
                    // Placeholder logic for touch
                    if (placeholder && targetTaskList) {
                        const childrenArray = Array.from(targetTaskList.children).filter(child => child !== placeholder && child !== touchDragElement);
                        let inserted = false;
                        for (let child of childrenArray) {
                            const childRect = child.getBoundingClientRect();
                            if (event.touches[0].clientY < childRect.top + childRect.height / 2) {
                                targetTaskList.insertBefore(placeholder, child);
                                inserted = true;
                                break;
                            }
                        }
                        if (!inserted) {
                            targetTaskList.appendChild(placeholder);
                        }
                    }
                }
            }
        }

        function handleTouchEnd(event) {
            if (!touchDragElement) return;

            document.removeEventListener('touchmove', handleTouchMove);
            document.removeEventListener('touchend', handleTouchEnd);

            if (touchClone) {
                touchClone.style.display = 'none'; // Hide clone to get element underneath for drop
                const elementUnderTouch = document.elementFromPoint(event.changedTouches[0].clientX, event.changedTouches[0].clientY);
                
                let droppedInColumn = false;
                if (elementUnderTouch) {
                    const targetColumnElement = elementUnderTouch.closest('.kanban-column');
                    if (targetColumnElement) {
                        const targetTaskList = targetColumnElement.querySelector('.task-list');
                        const targetColumnId = targetColumnElement.dataset.columnId;
                        const taskId = touchDragElement.dataset.taskId;

                        if (targetTaskList && taskId) {
                            // Move the original element to the new column (before placeholder)
                            if (placeholder && placeholder.parentNode) {
                                targetTaskList.insertBefore(touchDragElement, placeholder);
                            } else {
                                targetTaskList.appendChild(touchDragElement);
                            }
                            updateTaskColumn(taskId, targetColumnId);
                            droppedInColumn = true;
                        }
                    }
                }
                
                // If not dropped in a valid column, or placeholder logic failed,
                // it might revert or stay. The current logic moves it.
                // If it was not dropped in a column, it remains in its original list (due to re-rendering)
                // unless explicitly handled. renderTasks() will place it based on its 'column' property.
            }

            if (touchClone && touchClone.parentNode) {
                touchClone.parentNode.removeChild(touchClone);
            }
            if (placeholder && placeholder.parentNode) {
                placeholder.parentNode.removeChild(placeholder);
            }
            
            touchDragElement.classList.remove('dragging');
            columns.forEach(colList => colList.closest('.kanban-column').classList.remove('drag-over'));

            touchDragElement = null;
            touchClone = null;
            placeholder = null;
            
            renderTasks(); // Ensure UI consistency after touch drag
        }


        // --- Event Listeners ---
        addTaskBtn.addEventListener('click', addTask);
        newTaskInput.addEventListener('keypress', (event) => {
            if (event.key === 'Enter') {
                addTask();
            }
        });

        columns.forEach(columnList => {
            // Using columnList as the event target for dragover and drop
            // because it's the direct container for tasks.
            columnList.addEventListener('dragover', handleDragOver);
            columnList.addEventListener('drop', handleDrop);
            // Also add to the parent .kanban-column for broader drop area
            columnList.closest('.kanban-column').addEventListener('dragover', handleDragOver);
            columnList.closest('.kanban-column').addEventListener('drop', handleDrop);
            columnList.closest('.kanban-column').addEventListener('dragleave', (e) => {
                 // Only remove drag-over if not hovering over a child task list or task card
                if (!e.currentTarget.contains(e.relatedTarget) && !e.relatedTarget?.closest('.kanban-column')) {
                    e.currentTarget.classList.remove('drag-over');
                }
            });
        });

        // --- Confetti ---
        let confettiParticles = [];
        const confettiColors = ['#f9a8d4', '#f472b6', '#ec4899', '#db2777', '#9d174d', '#a78bfa', '#8b5cf6', '#7c3aed', '#6d28d9']; // Pinks and Purples

        function triggerConfetti() {
            confettiCanvas.style.display = 'block';
            confettiParticles = []; // Reset particles
            // Create a burst of particles
            for (let i = 0; i < 150; i++) { // More particles for a bigger effect
                confettiParticles.push(createConfettiParticle());
            }
            animateConfetti();

            // Hide confetti after a few seconds
            setTimeout(() => {
                confettiCanvas.style.display = 'none';
            }, 4000); // Increased duration
        }

        function createConfettiParticle() {
            const x = Math.random() * confettiCanvas.width;
            const y = Math.random() * confettiCanvas.height * 0.2 - 50; // Start near top, some slightly off-screen
            const size = Math.random() * 10 + 5; // Size between 5 and 15
            const speedX = Math.random() * 10 - 5; // Horizontal speed
            const speedY = Math.random() * 5 + 5;  // Vertical speed (fall downwards)
            const color = confettiColors[Math.floor(Math.random() * confettiColors.length)];
            const rotation = Math.random() * 360;
            const rotationSpeed = Math.random() * 20 - 10;

            return { x, y, size, speedX, speedY, color, rotation, rotationSpeed, opacity: 1 };
        }

        function animateConfetti() {
            if (confettiCanvas.style.display === 'none') return; // Stop if hidden

            confettiCtx.clearRect(0, 0, confettiCanvas.width, confettiCanvas.height);

            confettiParticles.forEach((p, index) => {
                p.x += p.speedX;
                p.y += p.speedY;
                p.rotation += p.rotationSpeed;
                p.opacity -= 0.005; // Fade out slowly

                // Apply slight "gravity" and "air resistance"
                p.speedY += 0.1; // Gravity
                p.speedX *= 0.99; // Air resistance for horizontal movement

                if (p.y > confettiCanvas.height || p.opacity <= 0) {
                    // Remove particle if it's off-screen or faded
                    confettiParticles.splice(index, 1);
                } else {
                    confettiCtx.save();
                    confettiCtx.translate(p.x + p.size / 2, p.y + p.size / 2);
                    confettiCtx.rotate(p.rotation * Math.PI / 180);
                    confettiCtx.fillStyle = p.color;
                    confettiCtx.globalAlpha = p.opacity;
                    // Draw a simple rectangle for confetti piece
                    confettiCtx.fillRect(-p.size / 2, -p.size / 2, p.size, p.size * 1.5); // Rectangular confetti
                    // confettiCtx.beginPath(); // For circular confetti
                    // confettiCtx.arc(0, 0, p.size / 2, 0, Math.PI * 2);
                    // confettiCtx.fill();
                    confettiCtx.restore();
                }
            });

            if (confettiParticles.length > 0) {
                requestAnimationFrame(animateConfetti);
            } else {
                confettiCanvas.style.display = 'none'; // Hide when all particles are gone
            }
        }

        function resizeConfettiCanvas() {
            confettiCanvas.width = window.innerWidth;
            confettiCanvas.height = window.innerHeight;
        }
        window.addEventListener('resize', resizeConfettiCanvas);


        // --- Local Storage ---
        const TASKS_STORAGE_KEY = 'kanbanTasksApp';

        function saveTasksToLocalStorage() {
            localStorage.setItem(TASKS_STORAGE_KEY, JSON.stringify(tasks));
        }

        function loadTasksFromLocalStorage() {
            const storedTasks = localStorage.getItem(TASKS_STORAGE_KEY);
            if (storedTasks) {
                tasks = JSON.parse(storedTasks);
            } else {
                // Add some default tasks if local storage is empty
                tasks = [
                    { id: generateId(), text: 'Brainstorm new project ideas', column: 'todo' },
                    { id: generateId(), text: 'Design homepage mockups', column: 'todo' },
                    { id: generateId(), text: 'Develop API endpoints', column: 'inprogress' },
                    { id: generateId(), text: 'User testing session', column: 'done' }
                ];
            }
        }

        // --- Initialization ---
        function init() {
            loadTasksFromLocalStorage();
            renderTasks();
            resizeConfettiCanvas(); // Set initial canvas size
        }

        init(); // Initialize the app
    </script>
</body>
</html>
