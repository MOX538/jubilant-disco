// No direct imports like Python, dependencies are handled via HTML script tags or JS modules.
// We will use the browser's built-in features:
// - Math object for math functions
// - Date.now() for time
// - Math.random() for random numbers
// - HTML5 Canvas API for graphics and event handling
// - HTML Audio API for sound

// Get canvas and context
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// Screen dimensions (set canvas size)
const WIDTH = 1920;
const HEIGHT = 1080;
canvas.width = WIDTH;
canvas.height = HEIGHT;

// Colors (using CSS color strings)
const WHITE = "rgb(255, 255, 255)";
const BLACK = "rgb(0, 0, 0)";
const GREEN = "rgb(0, 255, 0)";
const YELLOW = "rgb(255, 255, 0)";
const RED = "rgb(255, 0, 0)";
const BLUE = "rgb(0, 0, 255)";

// Fonts (using Canvas font syntax)
// Note: Font loading might need specific handling in a real application
// Using a generic 'Arial' as a fallback if 'None' isn't available/meant default.
let font = "60px Arial"; // Pygame's None usually means default system font

// Player properties
let player_radius = 25;
let player_pos = [WIDTH / 2, HEIGHT - player_radius * 2]; // Use floating point division
let player_speed = 20;
let player_shape = "circle"; // Default shape

// Bullet properties
let bullet_size = 10;
let bullets = []; // Array of [x, y] pairs
let bullet_speed = -50;

// Enemy properties
let enemy_size = 50;
let enemies = []; // List of enemies: {pos: [x, y], health: number}
let enemy_base_speed = 2; // Lower base speed

// Game properties
let lives = 3;
let max_health = 100;
let current_health = max_health;
let score = 0;
let difficulty_level = 1;
let spawn_rate = 30;
let start_time = Date.now() / 1000; // Use seconds like time.time()

// Clock (using requestAnimationFrame for game loop timing)
let lastTime = 0;
let gameLoopId = null; // To store the requestAnimationFrame ID

// Sound effects
let shoot_sound = null;
let hit_sound = null;
let explosion_sound = null;
try {
    shoot_sound = new Audio("shoot.wav"); // Assumes files are in the same directory
    hit_sound = new Audio("hit.wav");
    explosion_sound = new Audio("explosion.wav");
} catch (e) {
    console.error("Error loading sound files:", e);
    shoot_sound = hit_sound = explosion_sound = { play: () => {} }; // Dummy object to prevent errors
}

// --- Global state for input and mouse ---
let keysPressed = {};
let mousePos = { x: 0, y: 0 };
let mouseClicked = false;
let mouseWasClicked = false; // To detect single click edge

// --- Event Listeners ---
window.addEventListener('keydown', (e) => {
    keysPressed[e.code] = true; // Use e.code for layout-independent keys
});

window.addEventListener('keyup', (e) => {
    keysPressed[e.code] = false;
});

canvas.addEventListener('mousemove', (e) => {
    const rect = canvas.getBoundingClientRect();
    mousePos.x = e.clientX - rect.left;
    mousePos.y = e.clientY - rect.top;
});

canvas.addEventListener('mousedown', (e) => {
    if (e.button === 0) { // Left mouse button
        mouseClicked = true;
    }
});

canvas.addEventListener('mouseup', (e) => {
     if (e.button === 0) { // Left mouse button
        mouseClicked = false;
     }
});

// Helper to simulate pygame.quit()
function quitGame() {
    console.log("Quit action triggered.");
    // In a browser context, we can't reliably close the window/tab.
    // We can stop the game loop and maybe display a message.
    if (gameLoopId) {
        cancelAnimationFrame(gameLoopId);
        gameLoopId = null;
    }
    ctx.fillStyle = WHITE;
    ctx.fillRect(0, 0, WIDTH, HEIGHT);
    ctx.fillStyle = BLACK;
    ctx.font = "80px Arial";
    ctx.textAlign = "center";
    ctx.fillText("Game Quit", WIDTH / 2, HEIGHT / 2);
    // Remove event listeners if necessary, though stopping the loop is often enough
    window.removeEventListener('keydown', arguments.callee); // Example, needs proper handler reference
    window.removeEventListener('keyup', arguments.callee);
    // etc.
}


// Drawing shapes
function draw_shape(shape, surface_ctx, color, pos, size) {
    const x = pos[0];
    const y = pos[1];
    surface_ctx.fillStyle = color;
    surface_ctx.strokeStyle = color; // Some shapes might need stroke
    surface_ctx.beginPath();

    if (shape === "circle") {
        surface_ctx.arc(x, y, size, 0, Math.PI * 2);
        surface_ctx.fill();
    } else if (shape === "square") {
        surface_ctx.fillRect(x - size, y - size, size * 2, size * 2);
    } else if (shape === "triangle") {
        surface_ctx.moveTo(x, y - size);
        surface_ctx.lineTo(x - size, y + size);
        surface_ctx.lineTo(x + size, y + size);
        surface_ctx.closePath();
        surface_ctx.fill();
    } else if (shape === "diamond") {
        surface_ctx.moveTo(x, y - size);
        surface_ctx.lineTo(x - size, y);
        surface_ctx.lineTo(x, y + size);
        surface_ctx.lineTo(x + size, y);
        surface_ctx.closePath();
        surface_ctx.fill();
    } else if (shape === "hexagon") {
        for (let i = 0; i < 6; i++) {
            const angle = (Math.PI / 180) * (60 * i); // Convert degrees to radians
            const px = x + size * Math.cos(angle);
            const py = y + size * Math.sin(angle);
            if (i === 0) surface_ctx.moveTo(px, py);
            else surface_ctx.lineTo(px, py);
        }
        surface_ctx.closePath();
        surface_ctx.fill();
    } else if (shape === "star") {
        surface_ctx.moveTo(x, y - size); // Start point adjustment might be needed depending on desired star orientation
        for (let i = 0; i < 10; i++) {
            const angle = (Math.PI / 180) * (i * 36); // Convert degrees to radians
            const r = (i % 2 === 0) ? size : size / 2;
            const px = x + r * Math.cos(angle - Math.PI / 2); // Adjust angle offset if needed
            const py = y + r * Math.sin(angle - Math.PI / 2);
             surface_ctx.lineTo(px, py);
        }
        surface_ctx.closePath();
        surface_ctx.fill();
    } else if (shape === "cross") {
        const w = Math.floor(size / 2); // Use Math.floor for integer division
        surface_ctx.fillRect(x - w, y - size, w * 2, size * 2);
        surface_ctx.fillRect(x - size, y - w, size * 2, w * 2);
    } else if (shape === "heart") {
        // Approximate heart shape using curves and lines or simpler shapes
        // This implementation matches the Python version using circles and a triangle
        const top_curve_radius = Math.floor(size / 2);
        const triangle_height = size;
        // Draw triangle part
        surface_ctx.beginPath();
        surface_ctx.moveTo(x, y + triangle_height);
        surface_ctx.lineTo(x - size, y);
        surface_ctx.lineTo(x + size, y);
        surface_ctx.closePath();
        surface_ctx.fill();
        // Draw top circles
        surface_ctx.beginPath();
        surface_ctx.arc(x - top_curve_radius, y, top_curve_radius, 0, Math.PI * 2);
        surface_ctx.fill();
        surface_ctx.beginPath();
        surface_ctx.arc(x + top_curve_radius, y, top_curve_radius, 0, Math.PI * 2);
        surface_ctx.fill();
    }
}

// --- State Management for Screens ---
let currentScreenUpdater = null; // Function to call in the game loop
let screenRunning = false; // Flag to control screen-specific loops

// Show instructions screen
function show_instructions() {
    screenRunning = true;
    currentScreenUpdater = () => {
        if (!screenRunning) return; // Exit if screen should close

        ctx.fillStyle = WHITE;
        ctx.fillRect(0, 0, WIDTH, HEIGHT);
        ctx.font = font; // Ensure font is set
        ctx.fillStyle = BLACK;
        ctx.textAlign = "center"; // Center text horizontally

        const instructions = [
            "Instructions:",
            "Use LEFT and RIGHT arrow keys to move.",
            "Hold SPACE to shoot.",
            "Don't let enemies escape or touch you.",
            "Each level increases difficulty every 10 seconds.",
            "Green = Easy, Yellow = Medium, Red = Hard",
            "Press ESC to return to menu."
        ];

        for (let i = 0; i < instructions.length; i++) {
            const line = instructions[i];
            // No direct equivalent for text.get_width() before rendering in standard Canvas.
            // We render centered.
            ctx.fillText(line, WIDTH / 2, 100 + i * 70); // Adjusted spacing
        }

        // Event handling specific to this screen
        if (keysPressed['Escape']) {
             keysPressed['Escape'] = false; // Consume the key press
             screenRunning = false; // Signal to stop this screen
             game_menu(); // Go back to menu
        }
         // Check for quit event (e.g., closing the window - harder to catch reliably)
         // The main loop handles the global quit event if needed.
    };
    if (!gameLoopId) { // Start the loop if not already running
       lastTime = performance.now();
       gameLoopId = requestAnimationFrame(gameLoop);
    }
}

// Health bar
function draw_health_bar() {
    const bar_width = 300;
    const bar_height = 25;
    const x = 20;
    const y = 20;
    const fill = (current_health / max_health) * bar_width;

    // Background red bar
    ctx.fillStyle = RED;
    ctx.fillRect(x, y, bar_width, bar_height);

    // Foreground green bar
    ctx.fillStyle = GREEN;
    ctx.fillRect(x, y, Math.max(0, fill), bar_height); // Ensure fill isn't negative

    // Lives text
    ctx.fillStyle = BLACK;
    ctx.font = font;
    ctx.textAlign = "left"; // Align text to the left
    ctx.textBaseline = "top"; // Align text from the top
    const health_text = `Lives: ${lives}`;
    ctx.fillText(health_text, x, y + 30);
}

// Score
function draw_score() {
    ctx.fillStyle = BLACK;
    ctx.font = font;
    ctx.textAlign = "right"; // Align text to the right
    ctx.textBaseline = "top"; // Align text from the top
    const score_text = `Score: ${score}`;
    ctx.fillText(score_text, WIDTH - 20, 20);
}

function draw_difficulty_level() {
    ctx.fillStyle = BLACK;
    ctx.font = font;
    ctx.textAlign = "center"; // Align text to the center
    ctx.textBaseline = "top"; // Align text from the top
    const level_text = `Level: ${difficulty_level}`;
    ctx.fillText(level_text, WIDTH / 2, 20);
}

// Buttons
let buttonClickCooldown = false; // Prevent rapid clicks due to holding mouse down
function draw_button(text, x, y, width, height, action = null) {
    const mouse_x = mousePos.x;
    const mouse_y = mousePos.y;
    let isHovering = false;

    if (x < mouse_x && mouse_x < x + width && y < mouse_y && mouse_y < y + height) {
        isHovering = true;
        ctx.fillStyle = YELLOW;
        // Check for click *edge* (transition from not clicked to clicked)
        if (mouseClicked && !mouseWasClicked && action && !buttonClickCooldown) {
            buttonClickCooldown = true; // Set cooldown flag
            action();
            // Simulate time.sleep(0.2) effect for button cooldown
            setTimeout(() => { buttonClickCooldown = false; }, 200);
        }
    } else {
        ctx.fillStyle = BLUE;
    }
    ctx.fillRect(x, y, width, height);

    // Draw text on button
    ctx.fillStyle = BLACK;
    ctx.font = font;
    ctx.textAlign = "center";
    ctx.textBaseline = "middle"; // Center text vertically
    // Measure text width for horizontal centering
    // const textMetrics = ctx.measureText(text); // Not needed if textAlign is center
    ctx.fillText(text, x + width / 2, y + height / 2);
}

// Shape select screen
function shape_selection_screen() {
    screenRunning = true;
    let selecting = true; // Use local variable instead of modifying screenRunning directly initially
    const shapes = ["circle", "square", "triangle", "diamond", "hexagon", "star", "cross", "heart"];
    const positions = shapes.map((_, i) => [150 + i * 200, 300]);
    let selected = false; // Track if a shape has been clicked

    // Action for the continue button
    function exit_selection_screen() {
        selecting = false; // Stop this screen's logic
        screenRunning = false; // Allow main game to start
        // The start_game function will call main_game after this returns
    }

    currentScreenUpdater = () => {
        if (!selecting) return; // Stop if selection is done

        ctx.fillStyle = WHITE;
        ctx.fillRect(0, 0, WIDTH, HEIGHT);
        ctx.font = font;
        ctx.fillStyle = BLACK;
        ctx.textAlign = "center";
        ctx.textBaseline = "top";

        const instruction = "Choose Shape:";
        ctx.fillText(instruction, WIDTH / 2, 100);

        const mouse_x = mousePos.x;
        const mouse_y = mousePos.y;
        const click = mouseClicked && !mouseWasClicked; // Detect click edge

        for (let i = 0; i < shapes.length; i++) {
            const shape = shapes[i];
            const pos = positions[i];
            const color = (player_shape === shape) ? RED : BLACK;

            // Draw the shape preview
            draw_shape(shape, ctx, color, pos, 40);

            // Draw the label (number) below the shape
            ctx.fillStyle = BLACK; // Ensure label color is black
            ctx.font = "40px Arial"; // Smaller font for label
            ctx.textAlign = "center";
            ctx.textBaseline = "top";
            const label = `${i + 1}`;
            ctx.fillText(label, pos[0], pos[1] + 60);

            // Check for click on the shape area
            const shape_rect_x = pos[0] - 50;
            const shape_rect_y = pos[1] - 50;
            const shape_rect_w = 100;
            const shape_rect_h = 100;
            // Optional: Draw bounding box for debugging
            // ctx.strokeStyle = 'grey';
            // ctx.strokeRect(shape_rect_x, shape_rect_y, shape_rect_w, shape_rect_h);

            if (mouse_x > shape_rect_x && mouse_x < shape_rect_x + shape_rect_w &&
                mouse_y > shape_rect_y && mouse_y < shape_rect_y + shape_rect_h &&
                click) {
                player_shape = shape;
                selected = true; // Mark that a selection has been made
            }
        }

        // Draw the "Continue" button only after a shape has been selected
        if (selected) {
            draw_button("Continue", WIDTH / 2 - 150, 600, 300, 60, exit_selection_screen);
        }

        // Reset click edge detection for next frame
        mouseWasClicked = mouseClicked;

        // Check for quit event (handled globally)
    };

     // Need to wait until selection is done before proceeding
     // This requires modifying the flow slightly compared to Python's blocking call
     // We'll use a promise or callback structure, or simply let the game loop handle the transition
     // For simplicity, start_game will call this, and this screen's updater will run.
     // When exit_selection_screen is called, it sets flags that the main loop or start_game checks.
     // The current implementation relies on start_game calling main_game *after* this screen finishes.
     // Let's ensure the loop continues running.
     if (!gameLoopId) {
        lastTime = performance.now();
        gameLoopId = requestAnimationFrame(gameLoop);
     }
}


// Game starter
function start_game() {
    // Reset game state variables
    bullets = [];
    enemies = [];
    lives = 3;
    current_health = max_health;
    score = 0;
    difficulty_level = 1;
    spawn_rate = 30;
    start_time = Date.now() / 1000; // Reset timer
    player_pos = [WIDTH / 2, HEIGHT - player_radius * 2]; // Reset player position

    // Transition to shape selection
    game_state = "shape_select";
    shape_selection_screen(); // This sets up the updater for the shape screen

    // We need to wait for shape selection to complete before starting main_game.
    // The shape selection screen's "Continue" button action (exit_selection_screen)
    // will set flags. We need a mechanism to transition *after* that.
    // Let's modify the flow: start_game sets the state, shape_selection runs,
    // its exit action sets the state to "play", and the main loop picks up main_game.

    // The call to main_game() is removed from here. The state transition handles it.
    // main_game(); // REMOVED
}

// Menu screen
function game_menu() {
    game_state = "menu"; // Ensure state is correct
    screenRunning = true; // Indicate a screen-specific update loop is active

    function playAction() {
        screenRunning = false; // Stop menu updates
        start_game(); // This will set state to shape_select first
    }
    function instructionsAction() {
        screenRunning = false; // Stop menu updates
        show_instructions(); // This sets up its own updater
    }
    function quitAction() {
        screenRunning = false; // Stop menu updates
        quitGame();
    }

    currentScreenUpdater = () => {
        if (!screenRunning || game_state !== "menu") return; // Only run if this screen is active

        ctx.fillStyle = WHITE;
        ctx.fillRect(0, 0, WIDTH, HEIGHT);
        ctx.font = font;
        ctx.fillStyle = BLACK;
        ctx.textAlign = "center";
        ctx.textBaseline = "middle";

        const title_text = "Main Menu";
        ctx.fillText(title_text, WIDTH / 2, 150);

        draw_button("Play", WIDTH / 2 - 150, 300, 300, 60, playAction);
        draw_button("Instructions", WIDTH / 2 - 150, 400, 300, 60, instructionsAction);
        draw_button("Quit", WIDTH / 2 - 150, 500, 300, 60, quitAction);

        // Reset click edge detection for next frame
        mouseWasClicked = mouseClicked;

        // Global quit check (e.g., closing tab) is harder to implement reliably here
    };

    if (!gameLoopId) { // Start the loop if not already running
       lastTime = performance.now();
       gameLoopId = requestAnimationFrame(gameLoop);
    }
}

// Game over screen
function game_over_screen() {
    game_state = "game_over"; // Set state
    screenRunning = true;

    function goToMenuAction() {
        screenRunning = false;
        game_state = "menu"; // Set state for the main loop to pick up
        game_menu(); // Directly transition to menu setup
    }
    function restartAction() {
        screenRunning = false;
        start_game(); // Restart the game process (will go to shape select)
    }
    function quitAction() {
        screenRunning = false;
        quitGame();
    }

    currentScreenUpdater = () => {
        if (!screenRunning || game_state !== "game_over") return;

        ctx.fillStyle = WHITE;
        ctx.fillRect(0, 0, WIDTH, HEIGHT);
        ctx.font = font;
        ctx.textAlign = "center";
        ctx.textBaseline = "middle";

        ctx.fillStyle = RED;
        const over_text = "Game Over!";
        ctx.fillText(over_text, WIDTH / 2, 250);

        ctx.fillStyle = BLACK;
        const score_text = `Final Score: ${score}`;
        ctx.fillText(score_text, WIDTH / 2, 320); // Adjusted position slightly

        draw_button("Restart", WIDTH / 2 - 150, 400, 300, 60, restartAction);
        draw_button("Main Menu", WIDTH / 2 - 150, 480, 300, 60, goToMenuAction);
        draw_button("Quit", WIDTH / 2 - 150, 560, 300, 60, quitAction);

        // Reset click edge detection for next frame
        mouseWasClicked = mouseClicked;
    };

     if (!gameLoopId) { // Start the loop if not already running
       lastTime = performance.now();
       gameLoopId = requestAnimationFrame(gameLoop);
    }
}

// Main game loop logic (to be called by the requestAnimationFrame loop)
let shoot_cooldown = 0;
const enemy_damage = 34; // Amount of health reduced per enemy

function main_game_update() {
    // This function contains the logic previously inside the while loop of main_game

    // Clear screen
    ctx.fillStyle = WHITE;
    ctx.fillRect(0, 0, WIDTH, HEIGHT);

    // Calculate elapsed time and difficulty
    const current_time_sec = Date.now() / 1000;
    const elapsed_time = current_time_sec - start_time;
    difficulty_level = Math.floor(elapsed_time / 10) + 1;
    spawn_rate = Math.max(5, 30 - difficulty_level * 2);

    // Handle player movement input
    if (keysPressed['ArrowLeft'] || keysPressed['KeyA']) { // Added WASD support example
        player_pos[0] -= player_speed;
    }
    if (keysPressed['ArrowRight'] || keysPressed['KeyD']) {
        player_pos[0] += player_speed;
    }

    // Clamp player position
    player_pos[0] = Math.max(player_radius, Math.min(WIDTH - player_radius, player_pos[0]));

    // Handle shooting input
    if ((keysPressed['Space']) && shoot_cooldown === 0) {
        bullets.push([player_pos[0], player_pos[1] - player_radius]);
        shoot_cooldown = 10; // Cooldown frames (adjust based on desired rate)
        if (shoot_sound) {
             // Play sound - consider resetting currentTime for rapid fire
             shoot_sound.currentTime = 0;
             shoot_sound.play().catch(e => console.log("Shoot sound play failed:", e));
        }
    }

    // Update shoot cooldown
    if (shoot_cooldown > 0) {
        shoot_cooldown -= 1;
    }

    // Update bullets
    // Filter out bullets that go off-screen and update positions
    bullets = bullets.filter(b => b[1] > 0);
    bullets.forEach(b => {
        b[1] += bullet_speed; // Note: bullet_speed is negative
    });

    // Spawn enemies
    if (Math.random() * (spawn_rate + 1) < 1) { // Equivalent to random.randint(0, spawn_rate) == 0
        const x = Math.random() * (WIDTH - enemy_size * 2) + enemy_size; // Random x within bounds
        enemies.push({ "pos": [x, -enemy_size], "health": 1 });
    }

    // Update enemies
    const enemies_to_keep = [];
    for (let i = 0; i < enemies.length; i++) {
        let enemy = enemies[i];
        let keepEnemy = true;

        // Move enemy
        enemy["pos"][1] += enemy_base_speed + difficulty_level * 0.1;

        // Check if enemy reached bottom
        if (enemy["pos"][1] > HEIGHT) {
            keepEnemy = false; // Mark for removal
            current_health -= enemy_damage;
            if (explosion_sound) {
                explosion_sound.currentTime = 0;
                explosion_sound.play().catch(e => console.log("Explosion sound play failed:", e));
            }
            if (current_health <= 0) {
                lives -= 1;
                if (lives <= 0) {
                    game_state = "game_over"; // Transition state
                    game_over_screen(); // Setup game over screen
                    return; // Stop current game update
                } else {
                    current_health = max_health; // Reset health for next life
                }
            }
        }

        if (keepEnemy) {
            enemies_to_keep.push(enemy);
        }
    }
    enemies = enemies_to_keep;


    // Check bullet-enemy collisions
    const bullets_to_keep = [];
    for (let i = 0; i < bullets.length; i++) {
        let bullet = bullets[i];
        let bullet_hit = false;
        const remaining_enemies = [];

        for (let j = 0; j < enemies.length; j++) {
            let enemy = enemies[j];
            // Calculate distance between bullet center and enemy center
            // Enemy position is top-left, so center is (pos[0] + enemy_size / 2, pos[1] + enemy_size / 2)
            const dx = bullet[0] - (enemy["pos"][0] + enemy_size / 2);
            const dy = bullet[1] - (enemy["pos"][1] + enemy_size / 2);
            const distance = Math.hypot(dx, dy); // Use Math.hypot

            if (distance < bullet_size + enemy_size / 2) {
                // Collision detected
                bullet_hit = true; // Mark bullet for removal
                score += 10;
                if (hit_sound) {
                    hit_sound.currentTime = 0;
                    hit_sound.play().catch(e => console.log("Hit sound play failed:", e));
                }
                // Don't add this enemy to remaining_enemies, effectively removing it
            } else {
                remaining_enemies.push(enemy); // Keep this enemy
            }
        }
        enemies = remaining_enemies; // Update enemy list

        if (!bullet_hit) {
            bullets_to_keep.push(bullet); // Keep bullet if it didn't hit anything
        }
    }
    bullets = bullets_to_keep;


    // --- Drawing ---

    // Draw player
    draw_shape(player_shape, ctx, BLACK, player_pos, player_radius);

    // Draw bullets
    ctx.fillStyle = RED;
    bullets.forEach(b => {
        ctx.beginPath();
        ctx.arc(Math.round(b[0]), Math.round(b[1]), bullet_size, 0, Math.PI * 2); // Use integer coords for drawing
        ctx.fill();
    });

    // Draw enemies
    enemies.forEach(enemy => {
        const enemy_color = difficulty_level < 5 ? GREEN : (difficulty_level < 10 ? YELLOW : RED);
        ctx.fillStyle = enemy_color;
        ctx.fillRect(enemy["pos"][0], enemy["pos"][1], enemy_size, enemy_size);
    });

    // Draw UI elements
    draw_health_bar();
    draw_score();
    draw_difficulty_level();

     // Reset click edge detection for next frame (if needed by main game, though unlikely)
     mouseWasClicked = mouseClicked;
}

// The main animation loop function
function gameLoop(timestamp) {
    // Calculate time delta (optional, but good practice for physics/animation)
    const deltaTime = (timestamp - lastTime) / 1000; // Delta time in seconds
    lastTime = timestamp;

    // Clear the entire canvas (might be redundant if screen updaters always fill)
    // ctx.clearRect(0, 0, WIDTH, HEIGHT);

    // Execute the current screen's update logic
    if (game_state === "menu") {
        if (currentScreenUpdater) currentScreenUpdater();
    } else if (game_state === "shape_select") {
         if (currentScreenUpdater) currentScreenUpdater();
         // Check if selection is done and transition to play
         if (!screenRunning && game_state === "shape_select") { // screenRunning becomes false when Continue is clicked
             game_state = "play";
             // Reset player position just before starting main game logic
             player_pos = [WIDTH / 2, HEIGHT - player_radius * 2];
             // No need to call main_game_update here, next loop iteration will handle it
         }
    } else if (game_state === "instructions") {
        if (currentScreenUpdater) currentScreenUpdater();
         // Transition back to menu is handled within show_instructions via keypress
    } else if (game_state === "play") {
        main_game_update(); // Run the main game logic
    } else if (game_state === "game_over") {
        if (currentScreenUpdater) currentScreenUpdater();
        // Transitions handled by button clicks within game_over_screen
    }

    // Request the next frame
    if (game_state !== "quit") { // Check if quitGame was called
       gameLoopId = requestAnimationFrame(gameLoop);
    } else {
       // Optional: Final cleanup if needed after quitGame stops the loop
       console.log("Game loop stopped.");
    }
}

// Initial setup and start
let game_state = "menu"; // Start with the menu

// Entry point - start the menu
window.onload = () => {
    game_menu(); // Setup the menu screen updater and start the loop
};

