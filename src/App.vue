<template>
    <div class="app-wrapper">
        <!-- <header-vue></header-vue>
        <router-view/> -->
        <div class="wrapper">
            <div class="pong-container">
                <canvas ref="pongcanvas"
                        class="pong-canvas"></canvas>

                <div v-if="!gameIsRunning" class="menu">
                    <span v-on:click="selectDifficulty(menuItem)" v-for="menuItem in menuItems" :key="menuItem.id">{{menuItem.label}}</span>
                </div>
            </div>
        </div>

    </div>
</template>

<script>



    // import HeaderVue from "./components/header-vue";

    const INIT_BALL_SPEED = 0.25;
    const FAST_BALL_SPEED = 0.85;
    const SLOW_BALL_SPEED = 0.45;
    const MEDIUM_BALL_SPEED = 0.70;



    const FRAME_RATE = 60;
    const MS_PER_FRAME = 1000 / FRAME_RATE;
    const FRAME_PERCENTAGE_PER_SECOND = MS_PER_FRAME / 1000;

    const PADDLE_MOVEMENT_PER_SECOND_IN_PERCENTAGE = 0.5;


    class Vector2D {
        x;
        y;

        constructor(x, y) {
            this.x = x;
            this.y = y;
        }
    }

    class MenuItem {
        label;
        ballSpeed;
        id;

        constructor(label,id, ballSpeed) {
            this.id = id;
            this.label = label;
            this.ballSpeed = ballSpeed;
        }
    }

    class Paddle {
        position;
        height;
        width;

        constructor(position, height, width) {
            this.position = position;
            this.height = height;
            this.width = width;
        }

        positionLeftBottom() {
            return new Vector2D(this.position.x, this.position.y + this.height);
        }

        positionRightTop() {
            return new Vector2D(this.position.x + this.width, this.position.y);
        }

        positionRightBottom() {
            return new Vector2D(this.position.x + this.width, this.position.y + this.height);
        }

        centerPoint() {
            return new Vector2D(this.position.x + (this.width / 2), this.position.y + (this.height / 2));
        }
    }

    class Ball {
        ballDirection;
        ballPosition;
        ballSize;
        speed;

        constructor(ballPosition, ballSize, ballDirection, speed) {
            this.ballPosition = ballPosition;
            this.ballSize = ballSize;
            this.ballDirection = ballDirection;
            this.speed = speed
        }

        positionLeftBottom() {
            return new Vector2D(this.ballPosition.x, this.ballPosition.y + this.ballSize);
        }

        positionRightTop() {
            return new Vector2D(this.ballPosition.x + this.ballSize, this.ballPosition.y);
        }

        positionRightBottom() {
            return new Vector2D(this.ballPosition.x + this.ballSize, this.ballPosition.y + this.ballSize);
        }

        centerPoint() {
            return new Vector2D(this.ballPosition.x + (this.ballSize / 2), this.ballPosition.y + (this.ballSize / 2));
        }
    }

    export default {
        name: 'App',
        components: {
            // HeaderVue,
        },
        created() {
            window.addEventListener('keydown', this.onKeyDown);
            window.addEventListener('keyup', this.onKeyUp);
        },
        mounted() {
            this.pongCanvas = this.$refs.pongcanvas;
            this.drawingContext2D = this.pongCanvas.getContext("2d");
            this.drawingContext2D.scale(1, 1);
            this.pongCanvas.width = 750;
            this.pongCanvas.height = 500;

            this.paddleDimension.width = 15;
            this.paddleDimension.height = this.pongCanvas.height * 0.18;

            this.player1 = {
                paddle: new Paddle({
                    x: this.paddleDimension.width,
                    y: (this.pongCanvas.height / 2) - (this.paddleDimension.height / 2)
                }, this.paddleDimension.height, this.paddleDimension.width),
                score: 0,
                direction: 0,
            };

            this.player2 = {
                paddle: new Paddle({
                    x: this.pongCanvas.width - (2 * this.paddleDimension.width),
                    y: (this.pongCanvas.height / 2) - (this.paddleDimension.height / 2)
                }, this.paddleDimension.height, this.paddleDimension.width),
                score: 0,
                direction: 0,
            };

            this.resetBall();
            this.gameLoop();

        },
        methods: {
            resetBall() {
                const ballPosition = {
                    x: (this.pongCanvas.width / 2) - (this.ballSize / 2),
                    y: (this.pongCanvas.height / 2) - (this.ballSize / 2)
                };

                this.ball = new Ball(ballPosition, this.ballSize, this.calculateInitialBallDirection(), INIT_BALL_SPEED);
            },
            onKeyDown(event) {
                const pushedKey = event.key;
                if (!this.gameIsRunning) {
                    return;
                }
                if (this.pushedKeys.includes(pushedKey)) {
                    return;
                }

                this.pushedKeys.push(pushedKey);
            },
            onKeyUp(event) {

                const releasedKey = event.key;

                if (releasedKey === 'p') {
                    this.gameIsRunning = !this.gameIsRunning;
                    return;
                }

                if (releasedKey === 'r') {
                    this.resetBall();
                    return;
                }


                if (!this.gameIsRunning) {
                    return;
                }
                this.pushedKeys = this.pushedKeys.filter((pushedKey) => pushedKey !== releasedKey);
            },
            hasCollidedWithTopOrBottom() {
                return this.ball.ballPosition.y <= 0 || this.ball.ballPosition.y + this.ball.ballSize >= this.pongCanvas.height
            },
            hasPaddlePlayer1Collision() {
                return this.isValueBetween(this.player1.paddle.position.x, this.player1.paddle.positionRightTop().x, this.ball.ballPosition.x) &&
                    (
                        this.isValueBetween(this.player1.paddle.position.y, this.player1.paddle.positionRightBottom().y, this.ball.ballPosition.y) ||
                        this.isValueBetween(this.player1.paddle.position.y, this.player1.paddle.positionRightBottom().y, this.ball.positionLeftBottom().y)
                    );
            },
            hasPaddlePlayer2Collision() {
                return this.isValueBetween(this.player2.paddle.position.x, this.player2.paddle.positionRightTop().x, this.ball.positionRightTop().x) &&
                    (
                        this.isValueBetween(this.player2.paddle.position.y, this.player2.paddle.positionLeftBottom().y, this.ball.positionRightTop().y) ||
                        this.isValueBetween(this.player2.paddle.position.y, this.player2.paddle.positionLeftBottom().y, this.ball.positionRightBottom().y)
                    );
            },

            calculateBallMovement() {
                if (this.hasCollidedWithTopOrBottom()) {
                    this.ball.ballDirection.y *= -1;
                }
                // LinksOben{1,1}, RechtsOben {2,1}, LinksUnten {1, 5}, RechtsUnten {2,5}

                //BallPos {3, 1}

                if (this.hasPaddlePlayer1Collision()) {
                    console.log("CollisionP1");
                    this.calculateBallDirectionAfterCollision(this.player1.paddle);
                }

                if (this.hasPaddlePlayer2Collision()) {
                    console.log("CollisionP2");
                    this.calculateBallDirectionAfterCollision(this.player2.paddle, -1);
                }

                this.ball.ballPosition.x +=  (this.ball.ballDirection.x * (this.ball.speed * this.pongCanvas.width)) * FRAME_PERCENTAGE_PER_SECOND;
                this.ball.ballPosition.y += (this.ball.ballDirection.y * (this.ball.speed * this.pongCanvas.width)) * FRAME_PERCENTAGE_PER_SECOND;
                //todo
            },
            //1, 10, 5
            isValueBetween(start, end, value) {
                return value >= start && value <= end;
            },
            calculatePaddleMovement() {
                if (this.gameIsRunning) {
                    const paddleSpeedPerFrame = PADDLE_MOVEMENT_PER_SECOND_IN_PERCENTAGE * this.pongCanvas.height * FRAME_PERCENTAGE_PER_SECOND;
                    if (this.pushedKeys.includes('w')) {
                        this.player1.paddle.position.y = Math.max(0, this.player1.paddle.position.y - paddleSpeedPerFrame);
                    }
                    if (this.pushedKeys.includes('s')) {
                        this.player1.paddle.position.y = Math.min(500 - this.paddleDimension.height, this.player1.paddle.position.y + paddleSpeedPerFrame);
                    }


                    if (this.pushedKeys.includes('ArrowDown')) {
                        this.player2.paddle.position.y = Math.min(500 - this.paddleDimension.height, this.player2.paddle.position.y + paddleSpeedPerFrame);
                    }
                    if (this.pushedKeys.includes('ArrowUp')) {
                        this.player2.paddle.position.y = Math.max(0, this.player2.paddle.position.y - paddleSpeedPerFrame);
                    }
                }
            },
            drawRectangle(x, y, w, h, color = "white") {
                this.drawingContext2D.fillStyle = color;
                this.drawingContext2D.beginPath();
                this.drawingContext2D.fillRect(x, y, w, h);
                this.drawingContext2D.stroke();
            },
            drawBall() {
                this.drawRectangle(this.ball.ballPosition.x, this.ball.ballPosition.y, this.ball.ballSize, this.ball.ballSize);
            },
            drawPaddlePlayer1() {
                this.drawRectangle(this.player1.paddle.position.x, this.player1.paddle.position.y, this.paddleDimension.width, this.paddleDimension.height);
            },
            drawPaddlePlayer2() {
                this.drawRectangle(this.player2.paddle.position.x, this.player2.paddle.position.y, this.paddleDimension.width, this.paddleDimension.height);
            },
            clearCanvas() {
                this.drawingContext2D.clearRect(0, 0, this.pongCanvas.width, this.pongCanvas.height);
            },
            normalizeVector(vector2D) {
                const length = Math.sqrt(Math.pow(vector2D.x, 2) + Math.pow(vector2D.y, 2));
                vector2D.x = vector2D.x / length;
                vector2D.y = vector2D.y / length;
                return vector2D;
            },
            randomDirectionVector() {
                return {
                    x: 1,
                    y: Math.random() * 0.75
                };
            },
            randomizedKeySignature(vector2D) {
                if (Math.random() > 0.5) {
                    vector2D.x *= -1;
                }
                if (Math.random() > 0.5) {
                    vector2D.y *= -1;
                }

            },
            calculateInitialBallDirection() {
                const vector2D = this.randomDirectionVector();
                this.normalizeVector(vector2D);
                this.randomizedKeySignature(vector2D);
                return vector2D;
            },
            calculateBallDirectionAfterCollision(paddle, reflectDirection = 1) {
                const p1Center = paddle.centerPoint();
                const ballCenter = this.ball.centerPoint();
                const diffToPaddleCenter = p1Center.y - ballCenter.y; //Wenn Ergebnis Positiv ist der Ball in der Oberen Paddlearea. Wenn negativ dann unterhalb.
                const diffPercentage = Math.min(0.75, Math.abs(diffToPaddleCenter) / (paddle.height / 2 + this.ball.ballSize / 2));
                const yDirection = diffToPaddleCenter < 0 ? diffPercentage : diffPercentage * -1;
                const xDirection = reflectDirection * Math.sqrt(1 - Math.pow(yDirection, 2));
                this.ball.ballDirection = new Vector2D(xDirection, yDirection);
                this.ball.speed = this.selectedDifficulty.ballSpeed;
            },
            selectDifficulty(menuItem){
                this.selectedDifficulty = menuItem;
                this.gameIsRunning = true;
            },
            gameLoop() {
                if (this.gameIsRunning) {
                    this.clearCanvas();

                    this.calculatePaddleMovement();
                    this.calculateBallMovement();

                    // draw game components
                    this.drawBall();
                    this.drawPaddlePlayer1();
                    this.drawPaddlePlayer2();
                }

                setTimeout(this.gameLoop.bind(this), 1000 / 60)
            }
        },
        data() {
            return {
                menuItems: [new MenuItem("easy",1, SLOW_BALL_SPEED),new MenuItem("medium",2, MEDIUM_BALL_SPEED),new MenuItem("hard",3, FAST_BALL_SPEED)],
                selectedDifficulty: null,
                valorantLineupStore: null,
                running: false,
                pongCanvas: null,
                drawingContext2D: null,
                paddleDimension: {
                    width: 25,
                    height: 100
                },
                player1: {
                    paddle: null,
                    score: 0,
                },
                player2: {
                    paddle: null,
                    score: 0,
                },
                ball: null,
                ballDirection: {
                    x: 0,
                    y: 0,
                },
                gameIsRunning: false,
                pushedKeys: []
            }
        },
        computed: {
            ballSize: function () {
                return this.paddleDimension.width;
            }
        }
    }
</script>

<style scoped>
    body {
        font-family: 'Bahnschrift', sans-serif;
    }

    .app-wrapper {
        display: flex;
        flex-direction: column;
        align-items: center;
        height: 100vh;
    }


    .wrapper {
        position: fixed;
        width:100%;
        height:100vh;

        display: flex;
        align-items: center;
        justify-content: center;
    }
    .pong-container {
        display: flex;
        align-items: center;
        justify-content: center;
        padding: 10px 0;
        background-color: grey;
    }

    .pong-canvas {
        background-color: black;
    }

    .menu {
        position: absolute;
        color: white;
        display:flex;
        flex-direction: column;
        gap: 5px;
        text-align: center;

    }

    .menu span{
        padding: 5px;
    }

    .menu span:hover {
        background-color: white;
        color: black;
    }
</style>
