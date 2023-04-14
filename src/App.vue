<template>
    <div class="app-wrapper">
        <div class="wrapper">
            <div class="missileCommand-container">
                <canvas ref="missileCommandCanvas"
                        class="missileCommand-canvas"></canvas>
            </div>
        </div>

    </div>
</template>

<script>

    const FRAME_RATE = 60;
    const MS_PER_FRAME = 1000 / FRAME_RATE;
    // const FRAME_PERCENTAGE_PER_SECOND = MS_PER_FRAME / 1000;

    const MISSILE_FIRE_KEYS = ['a', 's', 'd'];
    const EXPLOSION_RADIUS_MAX = 0.035;

    class Vector2D {
        x;
        y;

        constructor(x, y) {
            this.x = x;
            this.y = y;
        }

        length() {
            return Math.sqrt(Math.pow(this.x, 2) + Math.pow(this.y, 2));
        }

        normalize() {
            const vectorLength = Math.sqrt(Math.pow(this.x, 2) + Math.pow(this.y, 2));
            this.x = this.x / vectorLength;
            this.y = this.y / vectorLength;
        }

        normalized() {
            const vectorLength = Math.sqrt(Math.pow(this.x, 2) + Math.pow(this.y, 2));
            return new Vector2D(this.x / vectorLength, this.y / vectorLength);
        }

        clone() {
            return Vector2D.from(this);
        }

        add(vector2D) {
            this.x += vector2D.x;
            this.y += vector2D.y;
        }

        multiply(value) {
            this.x *= value;
            this.y *= value;
        }

        multiplied(factor) {
            return new Vector2D(this.x * factor, this.y * factor);
        }

        equals(vector2D) {
            return this.x === vector2D.x && this.y === vector2D.y;
        }

        static from(vector2D) {
            return new Vector2D(vector2D.x, vector2D.y);
        }
    }


    class Town {
        x;
        y;
        width;
        height;
        centerTopPoint;
        destroyed;


        constructor(x, y, width, height, missileCommandCanvas) {
            this.x = x;
            this.y = y;
            this.width = width;
            this.height = height;
            this.centerTopPoint = new Vector2D(this.x + ((missileCommandCanvas.width * this.width) / 2), this.y);
        }
    }

    class TargetCross {
        leftPoint;
        rightPoint;
        bottomPoint;
        topPoint;

        constructor(centerPoint, crossLength) {
            this.leftPoint = new Vector2D(centerPoint.x - crossLength, centerPoint.y);
            this.rightPoint = new Vector2D(centerPoint.x + crossLength, centerPoint.y);
            this.bottomPoint = new Vector2D(centerPoint.x, centerPoint.y + crossLength);
            this.topPoint = new Vector2D(centerPoint.x, centerPoint.y - crossLength);
        }
    }

    class Missile {
        id;
        startingPoint;
        currentPoint;
        targetPoint;
        direction;
        readyToExplode = false;
        explosionRadius = 0;

        constructor(id, targetPoint) {
            this.id = id;
            this.targetPoint = targetPoint;
        }
    }

    class InterceptorMissile extends Missile {
        targetCross;

        constructor(id, targetPoint, targetCross) {
            super(id, targetPoint);
            this.targetCross = targetCross;
        }
    }


    class EnemyMissile extends Missile {

        constructor(id, startingPoint, targetPoint, direction) {
            super(id, targetPoint);
            this.startingPoint = startingPoint;
            this.currentPoint = startingPoint.clone();
            this.direction = direction;
        }
    }

    class Base extends Town {
        missileKey;


        constructor(x, y, width, height, missileKey, missileCommandCanvas) {
            super(x, y, width, height, missileCommandCanvas);
            this.missileKey = missileKey;
        }
    }

    const TOWN_WIDTH = 0.08;
    const TOWN_HEIGHT = 0.02;

    const BASE_WIDTH = 0.113;
    const GAP = 0.01;
    const BASE_HEIGHT = 0.04;

    const GROUND_POSITION_Y = 0.95;


    export default {
        name: 'App',
        components: {},
        created() {
            /*window.addEventListener('keydown', this.onKeyDown);
            window.addEventListener('keyup', this.onKeyUp);
            window.addEventListener("mousemove", this.onMouseMove);*/
        },
        mounted() {
            this.missileCommandCanvas = this.$refs.missileCommandCanvas;
            this.missileCommandCanvas.onclick = this.onMissileTargetClicked;
            window.onkeyup = this.onMissileFired;
            this.drawingContext2D = this.missileCommandCanvas.getContext("2d");
            this.drawingContext2D.scale(1, 1);

            this.calculateCanvasDimensions();
            this.generateTowns();
            this.generateBase();
            const missilesPerLevel = 100;
            this.maxMissiles = missilesPerLevel;
            this.remainingAvailableMissiles = missilesPerLevel;
            this.generateRandomEnemyMissiles();
            this.gameLoop();
        },
        //todo mouseevent für die Position auf dem Canvas , x zeichnen. √
        //todo jeder basis eine taste zuweisen. √
        //todo rakete abfeuern => linie zeichnen von basis bis einschlagspunkt √
        //todo am einschlagspunkt explosionsradius erzeugen ( kreis )
        methods: {
            drawTargetCross(targetCross) {
                this.drawLine(targetCross.leftPoint.x, targetCross.leftPoint.y, targetCross.rightPoint.x, targetCross.rightPoint.y);
                this.drawLine(targetCross.topPoint.x, targetCross.topPoint.y, targetCross.bottomPoint.x, targetCross.bottomPoint.y);
            },
            generateRandomEnemyMissiles() {
                const maxAttackingMissilesCount = 10;
                const maxAllowedAttackingMissiles = maxAttackingMissilesCount - this.generatedEnemyMissiles.length;
                const actuallyCreatableMissiles = Math.min(this.remainingAvailableMissiles, maxAllowedAttackingMissiles);
                const randomAmountOfMissiles = Math.floor(Math.random() * (actuallyCreatableMissiles + 1));
                //console.log(randomAmountOfMissiles);
                for (let i = 0; i < randomAmountOfMissiles; i++) {
                    this.generatedEnemyMissiles.push(this.generateRandomEnemyMissile(Date.now() + i));
                }
                //console.log("allowed " + actuallyCreatableMissiles);
                //console.log("created " + randomAmountOfMissiles);
                //console.log("remaining " + this.remainingAvailableMissiles);
                //console.log("");
                setTimeout(this.generateRandomEnemyMissiles, 500);
            },
            generateRandomEnemyMissile(id) {
                this.remainingAvailableMissiles -= 1;
                const targetPoint = this.getRandomTownOrBase().centerTopPoint.clone();
                const startingPoint = new Vector2D(this.missileCommandCanvas.width * Math.random(), 0);
                const directionVector = this.calculateDirectionVector(startingPoint, targetPoint);
                return new EnemyMissile(id, startingPoint, targetPoint, directionVector);
            },
            selectRandomTownForAttack() {
                const aliveTowns = this.towns.filter(town => !town.destroyed);
                return aliveTowns[this.getRandomIndex(aliveTowns.length)];
            },
            selectRandomBaseForAttack() {
                const aliveBases = this.bases.filter(base => !base.destroyed);
                return aliveBases[this.getRandomIndex(aliveBases.length)];
            },
            getRandomIndex(max) {
                return Math.floor(Math.random() * max)
            },
            getRandomTownOrBase() {

                // entscheide zuerst ob stadt oder basis, wenn eins von beiden nicht mehr geht dann das andere
                // wenn stadt dann nur städte die nicht zerstört sind
                // wenn basis dann nur basen die nicht zerstört sind
                if (this.bases.length === 0) {
                    return this.selectRandomTownForAttack()
                }

                if (this.towns.length === 0) {
                    return this.selectRandomBaseForAttack()
                }

                if ((this.isBaseSelected())) {
                    return this.selectRandomBaseForAttack();
                }
                return this.selectRandomTownForAttack();
            },
            isBaseSelected() {
                return (Math.random() < 0.4);
            },
            onMissileFired(keyboardEvent) {
                const pushedKey = keyboardEvent.key.toLowerCase();
                if (!MISSILE_FIRE_KEYS.includes(pushedKey)) {
                    return;
                }

                this.handlePlayerMissileFire(pushedKey);
            },
            handlePlayerMissileFire(pushedKey) {
                const responsibleBase = this.bases.find(base => base.missileKey === pushedKey);
                if (this.playerMissilesBeforeStart.length === 0) {
                    return;
                }

                const selectedMissile = this.selectNextNotFiredMissile();
                this.prepareMissileForLaunch(selectedMissile, responsibleBase);
            },
            selectNextNotFiredMissile() {
                let selectedMissile = this.playerMissilesBeforeStart[0];
                this.playerMissilesBeforeStart.forEach(notFiredMissile => {
                    if (notFiredMissile.missileCounter < selectedMissile.missileCounter) {
                        selectedMissile = notFiredMissile;
                    }
                });
                return selectedMissile;
            },
            prepareMissileForLaunch(selectedMissile, responsibleBase) {
                selectedMissile.startingPoint = responsibleBase.centerTopPoint.clone();
                selectedMissile.currentPoint = responsibleBase.centerTopPoint.clone();
                selectedMissile.direction = this.calculateDirectionVector(selectedMissile.startingPoint, selectedMissile.targetPoint);

                this.playerMissilesBeforeStart = this.playerMissilesBeforeStart.filter(missile => missile !== selectedMissile);
                this.playerMissiles.push(selectedMissile);
            },
            calculateDirectionVector(startingPoint, targetPoint) {
                const rawDirectionVector = new Vector2D(targetPoint.x - startingPoint.x, targetPoint.y - startingPoint.y);
                return rawDirectionVector.normalized();
            },
            onMissileTargetClicked(mouseEvent) {
                const targetPoint = new Vector2D(mouseEvent.offsetX, mouseEvent.offsetY);
                const targetCross = new TargetCross(targetPoint, this.missileCommandCanvas.width * 0.003);
                this.playerMissilesBeforeStart.push(new InterceptorMissile(this.missileCounter, targetPoint, targetCross));
                this.missileCounter += 1;
            },
            hasMissileReachedTarget(missile) {
                return missile.targetPoint.equals(missile.currentPoint);
            },
            hasEnemyMissileReachedTarget(enemyMissile) {
                return enemyMissile.targetPoint.y <= enemyMissile.currentPoint.y;
            },
            missileHasNotReachedTarget(currentDistanceOfMissileToTargetPoint, missile) {
                return currentDistanceOfMissileToTargetPoint > missile.direction.multiplied(this.dev_InterceptorMissileFactor).length()
            },
            calculatePlayerMissilesMovement() {
                const firedMissiles = this.playerMissiles.filter(missile => missile.startingPoint && missile.targetPoint && missile.direction && !missile.readyToExplode);
                firedMissiles.forEach(this.calculatePlayerMissileMovement);
            },
            calculatePlayerMissileMovement(missile) {
                const currentDistanceOfMissileToTargetVector = new Vector2D(missile.targetPoint.x - missile.currentPoint.x, missile.targetPoint.y - missile.currentPoint.y);
                if (this.hasMissileReachedTarget(missile)) {
                    missile.readyToExplode = true;
                    return;
                }

                if (this.missileHasNotReachedTarget(currentDistanceOfMissileToTargetVector.length(), missile)) {
                    missile.currentPoint.add(missile.direction.multiplied(this.dev_InterceptorMissileFactor));
                    return;
                }

                missile.currentPoint.add(currentDistanceOfMissileToTargetVector);
            },
            hasTownBeenHit() {
               const explodingEnemyMissiles = this.generatedEnemyMissiles.filter(missile => missile.readyToExplode);
               explodingEnemyMissiles.forEach(explodingEnemyMissile => {
                   this.towns.forEach(town => {
                       const distanceBetweenMissileAndTown = new Vector2D(Math.abs(explodingEnemyMissile.currentPoint.x - town.centerTopPoint.x),Math.abs(explodingEnemyMissile.currentPoint.y - town.centerTopPoint.y)).length();
                       if(distanceBetweenMissileAndTown <= explodingEnemyMissile.explosionRadius){
                           town.readyToExplode = true;
                           this.towns = this.towns.filter(town => !town.readyToExplode)
                       }
                   })
               })
           },
          hasBaseBeenHit() {
               const explodingEnemyMissiles = this.generatedEnemyMissiles.filter(missile => missile.readyToExplode);
               explodingEnemyMissiles.forEach(explodingEnemyMissile => {
                   this.bases.forEach(base => {
                       const distanceBetweenMissileAndTown = new Vector2D(Math.abs(explodingEnemyMissile.currentPoint.x - base.centerTopPoint.x),Math.abs(explodingEnemyMissile.currentPoint.y - base.centerTopPoint.y)).length();
                       if(distanceBetweenMissileAndTown <= explodingEnemyMissile.explosionRadius){
                           base.readyToExplode = true;
                           this.bases = this.bases.filter(base => !base.readyToExplode)
                       }
                   })
               })
           },
            isGameOver() {
                if(this.towns.length < 1) {
                    console.log("GAME OVER")
                }
            },
            detectMissileCollision() {
                const explodingPlayerMissiles = this.playerMissiles.filter(missile => missile.readyToExplode);
                explodingPlayerMissiles.forEach(explodingPlayerMissile => {
                    this.generatedEnemyMissiles.forEach(enemyMissile => {
                        const distanceBetweenMissiles = new Vector2D(Math.abs(enemyMissile.currentPoint.x - explodingPlayerMissile.targetPoint.x),Math.abs(enemyMissile.currentPoint.y - explodingPlayerMissile.targetPoint.y)).length();
                        if(distanceBetweenMissiles <= explodingPlayerMissile.explosionRadius){
                            //enemyMissile.readyToExplode = true; überlegen warum das so nicht geht
                            this.generatedEnemyMissiles = this.generatedEnemyMissiles.filter(missile => missile !== enemyMissile);
                        }
                    })
                });
            },
            calculateEnemyMissileMovement(enemyMissile) {
                if (this.hasEnemyMissileReachedTarget(enemyMissile)) {
                   enemyMissile.readyToExplode = true;
                    return
                }
                enemyMissile.currentPoint.add(enemyMissile.direction);
            },
            calculateEnemyMissilesMovement() {
                const firedEnemyMissiles = this.generatedEnemyMissiles.filter(enemyMissiles => !enemyMissiles.readyToExplode);
                firedEnemyMissiles.forEach(this.calculateEnemyMissileMovement)
            },
            drawMissiles(missiles, color) {
                const firedMissiles = missiles.filter(missile => !missile.readyToExplode);
                firedMissiles.forEach(missile => {
                    this.drawLine(missile.startingPoint.x, missile.startingPoint.y, missile.currentPoint.x, missile.currentPoint.y, color)
                });
            },
            drawRectangle(x, y, w, h, color = "white") {
                this.drawingContext2D.fillStyle = color;
                this.drawingContext2D.beginPath();
                this.drawingContext2D.fillRect(x, y, w, h);
                this.drawingContext2D.lineWidth = 15;
                this.drawingContext2D.stroke();
            },
            drawLine(startingX, startingY, endingX, endingY, color = "white") {
                this.drawingContext2D.beginPath();
                this.drawingContext2D.moveTo(startingX, startingY);
                this.drawingContext2D.lineTo(endingX, endingY);
                this.drawingContext2D.strokeStyle = color;
                this.drawingContext2D.lineWidth = 1;
                this.drawingContext2D.stroke();
            },
            drawCircle(center, radius, color = "white") {
                this.drawingContext2D.beginPath();
                this.drawingContext2D.arc(center.x, center.y, radius, 0, 2 * Math.PI, false);
                this.drawingContext2D.fillStyle = color;
                this.drawingContext2D.fill();
                this.drawingContext2D.lineWidth = 1;
                this.drawingContext2D.strokeStyle = color;
                this.drawingContext2D.stroke();
            },
            isMissileExplosionFinished(explodingMissile) {
                const maxExplosionRadiusPixel = EXPLOSION_RADIUS_MAX * this.missileCommandCanvas.width;
                return maxExplosionRadiusPixel < explodingMissile.explosionRadius;
            },
            calculateExplosionRadiuses(missiles) {
                const explodingMissiles = missiles.filter(m => m.readyToExplode);
                explodingMissiles
                    .forEach(explodingMissile => {
                            if (this.isMissileExplosionFinished(explodingMissile)) {
                                const explodedMissileIndex = missiles.indexOf(explodingMissile);
                                missiles.splice(explodedMissileIndex, 1);
                                return;
                            }
                            //still exploding
                            explodingMissile.explosionRadius += 1;
                        }
                    );
            },
            drawExplosions(missiles, color) {
                missiles
                    .filter(missile => missile.readyToExplode)
                    .forEach(missile => this.drawCircle(missile.targetPoint, missile.explosionRadius, color))
            },
            drawTargetCrosses() {
                this.playerMissilesBeforeStart.forEach(missile => this.drawTargetCross(missile.targetCross));
                this.playerMissiles
                    .filter(missile => !missile.readyToExplode)
                    .forEach(missile => this.drawTargetCross(missile.targetCross));
            },
            calculateCanvasDimensions() {
                this.missileCommandCanvas.width = window.innerWidth * 0.70;
                this.missileCommandCanvas.height = this.missileCommandCanvas.width * (2 / 3);
            },
            generateTowns() {
                const currentPosition = new Vector2D(this.missileCommandCanvas.width * (BASE_WIDTH + 3 * GAP), this.missileCommandCanvas.height * GROUND_POSITION_Y);
                const completeGap = (this.missileCommandCanvas.width * (TOWN_WIDTH + (2 * GAP)));
                for (let i = 1; i <= 6; i++) {
                    if (i === 4) {
                        currentPosition.x += this.missileCommandCanvas.width * (BASE_WIDTH + 2 * GAP);
                    }
                    const town = new Town(currentPosition.x, currentPosition.y, TOWN_WIDTH, TOWN_HEIGHT, this.missileCommandCanvas);
                    this.towns.push(town);
                    currentPosition.x += completeGap;
                }
            },
            generateBase() {
                const completeGap = (this.missileCommandCanvas.width * ((2 * GAP) + BASE_WIDTH + (3 * (TOWN_WIDTH + 2 * GAP))));
                const currentPosition = new Vector2D(this.missileCommandCanvas.width * GAP, this.missileCommandCanvas.height * GROUND_POSITION_Y);
                const base1 = new Base(currentPosition.x, currentPosition.y, BASE_WIDTH, BASE_HEIGHT, "a", this.missileCommandCanvas);
                currentPosition.x += completeGap;
                const base2 = new Base(currentPosition.x, currentPosition.y, BASE_WIDTH, BASE_HEIGHT, "s", this.missileCommandCanvas);
                currentPosition.x += completeGap;
                const base3 = new Base(currentPosition.x, currentPosition.y, BASE_WIDTH, BASE_HEIGHT, "d", this.missileCommandCanvas);
                this.bases = [base1, base2, base3];
            },
            drawTown(town) {
                this.drawRectangle(town.x, town.y, this.missileCommandCanvas.width * town.width, this.missileCommandCanvas.height * town.height, "white")
            },
            drawBase(base) {
                this.drawRectangle(base.x, base.y, this.missileCommandCanvas.width * base.width, this.missileCommandCanvas.height * base.height, "white")
            },
            clearCanvas() {
                this.drawingContext2D.clearRect(0, 0, this.missileCommandCanvas.width, this.missileCommandCanvas.height);
            },
            drawTowns() {
                this.towns.forEach(town => this.drawTown(town));
            },
            drawBases() {
                this.bases.forEach(base => this.drawBase(base));
            },
            gameLoop() {
                this.clearCanvas();

                this.calculatePlayerMissilesMovement();
                this.calculateEnemyMissilesMovement();
                this.calculateExplosionRadiuses(this.playerMissiles);
                this.calculateExplosionRadiuses(this.generatedEnemyMissiles);

                this.detectMissileCollision();

                this.hasTownBeenHit();
                this.hasBaseBeenHit();

                this.isGameOver();

                this.drawTowns();
                this.drawBases();
                this.drawExplosions(this.playerMissiles, "white");
                this.drawExplosions(this.generatedEnemyMissiles, "red");
                this.drawTargetCrosses();
                this.drawMissiles(this.playerMissiles, "white");
                this.drawMissiles(this.generatedEnemyMissiles, "red");
                setTimeout(this.gameLoop.bind(this), MS_PER_FRAME);
            },
        },
        data() {
            return {
                missileCommandCanvas: null,
                drawingContext2D: null,
                pxValue: 50,
                towns: [],
                bases: [],
                playerMissiles: [],
                playerMissilesBeforeStart: [],
                // playerMissilesGraveyard: [],
                missileCounter: 0,
                generatedEnemyMissiles: [],
                maxMissiles: 10,
                remainingAvailableMissiles: 10,
                dev_InterceptorMissileFactor: 100
            }
        },
        computed: {}
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
        width: 100%;
        height: 100vh;

        display: flex;
        align-items: center;
        justify-content: center;
    }

    .missileCommand-container {
        display: flex;
        align-items: center;
        justify-content: center;
        padding: 10px 0;
        background-color: grey;
    }

    .missileCommand-canvas {
        background-color: black;
    }
</style>
