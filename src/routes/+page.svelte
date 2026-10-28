
<script>
    import { onMount } from "svelte";

    let alpha = $state(null);
    let beta = $state(null);
    let gamma = $state(null);

    let diceNumber = $state(null);
    let status = $state("Tap Start, then shake your phone.");
    let error = $state("");

    let started = false;
    let previousAcceleration = null;
    let lastShakeTime = 0;

    const SHAKE_THRESHOLD = 15;
    const SHAKE_COOLDOWN = 1200;

    // Decide the number based on the phone's orientation.
    function getDiceNumber(beta, gamma) {
        if (beta === null || gamma === null) {
            return null;
        }

        // Phone facing down.
        if (Math.abs(beta) >= 135) {
            return 1;
        }

        // Phone held upright or tilted vertically.
        if (beta >= 45) {
            return 6;
        }

        // Phone tilted in the opposite vertical direction.
        if (beta <= -45) {
            return 2;
        }

        // Phone tilted to the left or right.
        if (gamma >= 45) {
            return 4;
        }

        if (gamma <= -45) {
            return 3;
        }

        // Phone lying flat with its screen facing up.
        return 5;
    }

    function handleOrientation(event) {
        alpha = event.alpha;
        beta = event.beta;
        gamma = event.gamma;
    }

    function handleMotion(event) {
        const acceleration = event.accelerationIncludingGravity;

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

        if (previousAcceleration !== null) {
            const dx = current.x - previousAcceleration.x;
            const dy = current.y - previousAcceleration.y;
            const dz = current.z - previousAcceleration.z;

            const movement = Math.sqrt(
                dx * dx + dy * dy + dz * dz
            );

            const now = Date.now();

            if (
                movement > SHAKE_THRESHOLD &&
                now - lastShakeTime > SHAKE_COOLDOWN
            ) {
                lastShakeTime = now;

                const number = getDiceNumber(beta, gamma);

                if (number !== null) {
                    diceNumber = number;
                    status = "Dice activated! Shake again to roll.";
                } else {
                    status = "Waiting for orientation data...";
                }
            }
        }

        previousAcceleration = current;
    }

    async function startDice() {
        error = "";

        try {
            // Some browsers require permission after a user tap.
            if (
                typeof DeviceOrientationEvent !== "undefined" &&
                typeof DeviceOrientationEvent.requestPermission === "function"
            ) {
                const permission =
                    await DeviceOrientationEvent.requestPermission();

                if (permission !== "granted") {
                    error = "Orientation permission was denied.";
                    return;
                }
            }

            if (
                typeof DeviceMotionEvent !== "undefined" &&
                typeof DeviceMotionEvent.requestPermission === "function"
            ) {
                const permission =
                    await DeviceMotionEvent.requestPermission();

                if (permission !== "granted") {
                    error = "Motion permission was denied.";
                    return;
                }
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

            started = true;
            status = "Shake your phone to activate the dice!";
        } catch (err) {
            error = "Could not access the sensors. Check browser permissions.";
        }
    }

    onMount(() => {
        return () => {
            window.removeEventListener(
                "deviceorientation",
                handleOrientation
            );

            window.removeEventListener(
                "devicemotion",
                handleMotion
            );
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

    {#if !started}
        <button onclick={startDice}>
            Start Dice
        </button>
    {/if}

    {#if error}
        <p class="error">{error}</p>
    {/if}

    <h2>Sensor values</h2>

    <p>Alpha: {alpha === null ? "--" : Math.round(alpha)}</p>
    <p>Beta: {beta === null ? "--" : Math.round(beta)}</p>
    <p>Gamma: {gamma === null ? "--" : Math.round(gamma)}</p>
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

    button {
        padding: 12px 24px;
        font-size: 16px;
        cursor: pointer;
    }

    .error {
        color: red;
    }
</style>
