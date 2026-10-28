
<script>
    import { onMount } from "svelte";

    let alpha = $state(null);
    let beta = $state(null);
    let gamma = $state(null);

    let diceNumber = $state(null);
    let status = $state("Waiting for motion...");
    let error = $state("");

    let sensorsStarted = false;
    let previousAcceleration = null;
    let lastShakeTime = 0;
    let shakePending = false;
    let rollTimer;

    const SHAKE_THRESHOLD = 15;
    const SHAKE_COOLDOWN = 1800;
    const WAIT_AFTER_SHAKE = 1000;

    // Read the direction using DeviceOrientationEvent.
    function handleOrientation(event) {
        alpha = event.alpha;
        beta = event.beta;
        gamma = event.gamma;
    }

    // Convert the current orientation into a fixed dice number.
    function getDiceNumber() {
        if (beta === null || gamma === null) {
            return null;
        }

        const pitch = beta;
        const roll = gamma;

        // Screen facing down.
        if (Math.abs(pitch) > 135) {
            return 1;
        }

        // Phone held upright in either direction.
        if (Math.abs(pitch) >= 45) {
            return pitch > 0 ? 6 : 2;
        }

        // Phone tilted onto either side.
        if (Math.abs(roll) >= 45) {
            return roll > 0 ? 4 : 3;
        }

        // Phone lying flat, screen facing up.
        return 5;
    }

    // Detect a shake using DeviceMotionEvent.
    function handleMotion(event) {
        const acceleration =
            event.accelerationIncludingGravity;

        if (
            !acceleration ||
            acceleration.x === null ||
            acceleration.y === null ||
            acceleration.z === null
        ) {
            return;
        }

        const current = {
            x: acceleration.x,
            y: acceleration.y,
            z: acceleration.z
        };

        if (previousAcceleration) {
            const dx = current.x - previousAcceleration.x;
            const dy = current.y - previousAcceleration.y;
            const dz = current.z - previousAcceleration.z;

            const movement = Math.sqrt(
                dx * dx + dy * dy + dz * dz
            );

            const now = Date.now();

            if (
                movement > SHAKE_THRESHOLD &&
                now - lastShakeTime > SHAKE_COOLDOWN &&
                !shakePending
            ) {
                lastShakeTime = now;
                shakePending = true;

                status = "Shake detected! Point your phone...";

                // Wait one second BEFORE reading the direction.
                clearTimeout(rollTimer);

                rollTimer = setTimeout(() => {
                    const result = getDiceNumber();

                    if (result === null) {
                        status = "No orientation data received.";
                    } else {
                        diceNumber = result;
                        status = "Result: " + result;
                    }

                    shakePending = false;
                }, WAIT_AFTER_SHAKE);
            }
        }

        previousAcceleration = current;
    }

    // Start listening to the sensors.
    async function startSensors() {
        if (sensorsStarted) return;

        error = "";

        try {
            const permissions = [];

            // iOS may require permission for both APIs.
            if (
                typeof DeviceOrientationEvent !== "undefined" &&
                typeof DeviceOrientationEvent.requestPermission === "function"
            ) {
                permissions.push(
                    DeviceOrientationEvent.requestPermission()
                );
            }

            if (
                typeof DeviceMotionEvent !== "undefined" &&
                typeof DeviceMotionEvent.requestPermission === "function"
            ) {
                permissions.push(
                    DeviceMotionEvent.requestPermission()
                );
            }

            const results = await Promise.all(permissions);

            if (results.some(result => result !== "granted")) {
                error = "Please allow motion and orientation access.";
                status = "Sensor permission required.";
                return;
            }

            if (
                typeof DeviceOrientationEvent === "undefined" ||
                typeof DeviceMotionEvent === "undefined"
            ) {
                error = "This browser does not support the required sensors.";
                return;
            }

            window.addEventListener(
                "deviceorientation",
                handleOrientation
            );

            window.addEventListener(
                "devicemotion",
                handleMotion
            );

            sensorsStarted = true;
            status = "Ready! Shake your phone.";
        } catch (err) {
            error = "Could not start sensors. Tap the page to try again.";
        }
    }

    onMount(() => {
        const needsPermission =
            (
                typeof DeviceOrientationEvent !== "undefined" &&
                typeof DeviceOrientationEvent.requestPermission === "function"
            ) ||
            (
                typeof DeviceMotionEvent !== "undefined" &&
                typeof DeviceMotionEvent.requestPermission === "function"
            );

        // Start automatically when permissions are not required.
        if (!needsPermission) {
            startSensors();
        }

        // On iPhones, use the first screen tap as the permission gesture.
        function handleFirstTap() {
            startSensors();
        }

        if (needsPermission) {
            window.addEventListener("pointerdown", handleFirstTap);
        }

        return () => {
            window.removeEventListener(
                "pointerdown",
                handleFirstTap
            );

            window.removeEventListener(
                "deviceorientation",
                handleOrientation
            );

            window.removeEventListener(
                "devicemotion",
                handleMotion
            );

            clearTimeout(rollTimer);
        };
    });
</script>

<svelte:head>
    <meta
        name="viewport"
        content="width=device-width, initial-scale=1"
    />
    <title>Orientation Dice</title>
</svelte:head>

<main>
    <h1>Orientation Dice</h1>

    <p>{status}</p>

    <div class="dice">
        {diceNumber ?? "?"}
    </div>

    {#if error}
        <p class="error">{error}</p>
    {/if}

    <section>
        <p>Alpha: {alpha === null ? "--" : Math.round(alpha)}°</p>
        <p>Beta: {beta === null ? "--" : Math.round(beta)}°</p>
        <p>Gamma: {gamma === null ? "--" : Math.round(gamma)}°</p>
    </section>
</main>

<style>
    main {
        font-family: Arial, sans-serif;
        text-align: center;
        padding: 24px;
    }

    .dice {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 150px;
        height: 150px;
        margin: 24px auto;
        border: 3px solid currentColor;
        border-radius: 24px;
        font-size: 72px;
        font-weight: bold;
    }

    .error {
        color: red;
    }
</style>
