<template>
  <img ref="bugImg" :src="imgSrc" alt=""/>
</template>

<script>
import {distance, plusOrMinus, randBM, randomRange} from '@/js/helpers'

export default {
  name: 'bug',
  data() {
    return {
      bugTypeOptions: require('@/config.json')['bugTypes'],
      targetDriftsOptions: require('@/config.json')['targetDrifts'],
      bugImages: [],
      currentBugType: undefined,
      currentBugSize: undefined,
      imgSrc: '',
      mass: 1,
      randomNoise: 0,
      frameCounter: 0
    }
  },
  props: {
    x0: Number,
    y0: Number,
    bugsSettings: Object
  },
  computed: {
    isMoveInCircles: function () {
      return this.bugsSettings.movementType === 'circle'
    },
    isStraightLine: function () {
      return this.bugsSettings.movementType === 'random'
    },
    isRandomDrift: function () {
      return this.bugsSettings.movementType === 'random_drift'
    },
    isLowHorizontal: function () {
      return this.bugsSettings.movementType === 'low_horizontal'
    },
    isNoisyLowHorizontal: function () {
      return this.bugsSettings.movementType === 'low_horizontal_noise'
    },
    isStatic: function () {
      return this.bugsSettings.movementType === 'static'
    },
    bugHeight: function () {
      return this.bugsSettings.bugHeight ? Number(this.bugsSettings.bugHeight) : 100
    },
    numImagesPerBug: function () {
      return this.bugTypeOptions[this.currentBugType].numImagesPerBug
    },
    stepsPerImage: function () {
      return this.bugTypeOptions[this.currentBugType].stepsPerImage
    },
    currentSpeed: function () {
      if (this.bugsSettings && this.bugsSettings.speed) {
        return this.bugsSettings.speed
      }
      return this.bugTypeOptions[this.currentBugType].speed
    },
    noiseStartX: function () {
      let halfScreen = this.canvas.width / 2
      if (this.xTarget > halfScreen) {
        return this.canvas.width * this.bugsSettings.noiseStartFraction
      } else {
        return this.canvas.width * (1 - this.bugsSettings.noiseStartFraction)
      }
    }
  },
  mounted() {
    this.canvas = this.$parent.canvas
    this.ctx = this.canvas.getContext('2d')
    this.isOutEdged = false
    this.isDead = false
    if (!Array.isArray(this.bugsSettings.bugTypes)) {
      this.bugsSettings.bugTypes = [this.bugsSettings.bugTypes]
    }
    if (this.isMoveInCircles) {
      this.r0 = [this.canvas.width / 2, this.canvas.height / 2]
      this.r = Math.min(this.canvas.width / 2, this.canvas.height / 2) * 0.75
    } else if (this.isRandomDrift || this.isLowHorizontal || this.isNoisyLowHorizontal) {
      this.initTargetDrift()
    }
    this.initBug(this.x, this.y)
  },
  methods: {
    initBug(x, y) {
      this.frameCounter = 0
      this.getNextBugType()
      if (!this.isMoveInCircles) {
        this.initSpeedDirections()
        if (this.isRandomDrift || this.isStraightLine) {
          this.startRandomEdgePoint()
        } else if (this.isLowHorizontal || this.isNoisyLowHorizontal) {
          this.startHorizontal()
        }
      }
      this.step = 0
      this.randomNoiseCount = 0
      this.theta = Math.PI - Math.PI / 20
      this.currentBugSize = this.getRadiusSize()
    },
    move(particles) {
      if (this.isDead || this.isOutEdged) {
        this.draw()
        return
      }
      this.frameCounter++
      this.edgeDetection()
      if (this.isMoveInCircles) {
        // Circular Movement
        this.theta += Math.abs(this.currentSpeed) * Math.sqrt(2) / this.r
        this.x = this.r0[0] + (this.r * Math.cos(this.theta)) * (this.bugsSettings.isAntiClockWise ? -1 : 1)
        this.y = this.r0[1] + this.r * Math.sin(this.theta)
      } else if (this.isStatic) {
        this.x = this.canvas.width / 2
        this.y = this.canvas.height / 2
        this.dx = 0
        this.dy = 0
      } else {
        let randNoise = this.getRandomNoise()
        let halfScreen = this.canvas.width
        if (this.isNoisyLowHorizontal &&
            ((this.xTarget < halfScreen && this.x < this.noiseStartX) ||
                (this.xTarget > halfScreen && this.x > this.noiseStartX))) {
          // low horizontal + Noise (The noisy part)
          this.dx = 0.5 * this.vx + 0.5 * randNoise
          this.dy = 0.0008 * (this.yTarget - this.y) + 0.5 * randNoise + 0.9 * this.dy
        } else if (this.isRandomDrift && this.frameCounter > 100) {
          // random drift after 100 frames
          this.dx = 0.004 * (this.xTarget - this.x) + 0.5 * randNoise
          this.dy = 0.004 * (this.yTarget - this.y) + 0.5 * randNoise
        } else if (this.isStraightLine || this.isRandomDrift) {
          // straight lines (random walk) w/ little gaussian noise
          this.dx = this.vx + 0.5 * randNoise
          this.dy = this.vy + 0.5 * randNoise
        }
        this.x += this.dx
        this.y += this.dy
      }
      this.draw()

      for (let i = 0; i < particles.length; i++) {
        if (this === particles[i]) continue

        if (distance(this.x, this.y, particles[i].x, particles[i].y) <= this.currentBugSize + particles[i].currentRadiusSize) {
          this.collisionEffect(particles[i])
        }
      }
    },
    getRandomNoise() {
      if (this.randomNoiseCount > 20) {
        this.randomNoiseCount = 0
        this.randomNoise = randBM()
      }
      this.randomNoiseCount++
      return this.randomNoise
    },
    draw() {
      this.ctx.beginPath()
      let imgIndex = Math.floor(this.step / this.stepsPerImage)
      this.imgSrc = this.getImageSrc(`/${this.currentBugType}${imgIndex}.png`)
      this.drawImage()
      this.step++
      if (this.step > (this.numImagesPerBug - 1) * this.stepsPerImage) {
        this.step = 0
      }
      this.ctx.fill()
      this.ctx.closePath()
    },
    drawImage() {
      try {
        let bugImage = this.isDead ? this.getDeadImage() : this.$refs.bugImg
        this.ctx.setTransform(1, 0, 0, 1, this.x, this.y)
        this.ctx.rotate(this.getAngleRadians())
        // drawImage(image, dx, dy, dWidth, dHeight)
        this.ctx.drawImage(bugImage, -this.currentBugSize / 2, -this.currentBugSize / 2, this.currentBugSize, this.currentBugSize)
        this.ctx.setTransform(1, 0, 0, 1, 0, 0)
        // this.ctx.arc(this.x, this.y, this.currentRadiusSize / 1.5, 0, 2 * Math.PI, false)
        // this.ctx.fillStyle = 'rgba(255,0,0,0.3)'
        // this.ctx.fill()
      } catch (e) {
        console.error(e)
      }
    },
    edgeTimeout() {
      this.isOutEdged = true
      const initAfterEdgeTimeout = setTimeout(() => {
        this.initBug(this.xEdge(), this.yEdge())
        const outEdgeTimeout = setTimeout(() => {
          this.isOutEdged = false
          clearTimeout(outEdgeTimeout)
        }, 50)
        clearTimeout(initAfterEdgeTimeout)
      }, this.bugsSettings.timeInEdge)
    },
    edgeDetection() {
      if (this.isOutEdged) {
        return
      }
      let isHorizontal = this.isLowHorizontal || this.isNoisyLowHorizontal
      if ((isHorizontal && (this.x < -this.currentBugSize || this.x > this.canvas.width + this.currentBugSize)) ||
          (this.isRandomDrift && distance(this.x, this.xTarget, this.y, this.yTarget) < this.currentBugSize)) {
        this.edgeTimeout()
        return
      }
      if (!isHorizontal) {
        const margin = this.currentBugSize / 2
        if ((this.x >= this.canvas.width + margin || this.x <= -margin) ||
            (this.y >= this.canvas.height + margin || this.y <= -margin)) {
          this.edgeTimeout()
        }
      }
    },
    getImageSrc(fileName) {
      return require('@/assets' + fileName)
    },
    getDeadImage() {
      let img = new Image()
      img.src = this.getImageSrc(`/${this.currentBugType}_dead.png`)
      return img
    },
    getNextBugType() {
      if (this.bugsSettings.bugTypes.length === 1) {
        this.currentBugType = this.bugsSettings.bugTypes[0]
        return
      }
      let nextBugOptions = this.bugsSettings.bugTypes.filter(bug => bug !== this.currentBugType)
      let nextIndex = randomRange(0, nextBugOptions.length)
      this.currentBugType = nextBugOptions[nextIndex]
    },
    getRadiusSize() {
      if (this.bugsSettings.bugSize) {
        return this.bugsSettings.bugSize
      }
      let currentBugOptions = this.bugTypeOptions[this.currentBugType]
      return randomRange(currentBugOptions.radiusRange.min, currentBugOptions.radiusRange.max)
    },
    xEdge() {
      if (this.isMoveInCircles) {
        return this.r0[0] - Math.sqrt(this.r ** 2 - (this.canvas.height - this.r0[1]) ** 2)
      }
      if (this.x > this.canvas.width) {
        return this.canvas.width
      } else if (this.x < 0) {
        return 0
      }
      return this.x
    },
    yEdge() {
      if (this.y > this.canvas.height) {
        return this.canvas.height
      } else if (this.y < 0) {
        return 0
      }
      return this.y
    },
    startRandomEdgePoint() {
      let rand = Math.random()
      if (rand < 0.25) {
        this.x = 0
        this.y = randomRange(0, this.canvas.height)
      } else if (rand >= 0.25 && rand < 0.5) {
        this.x = this.canvas.width
        this.y = randomRange(0, this.canvas.height)
      } else if (rand >= 0.5 && rand < 0.75) {
        this.x = randomRange(0, this.canvas.width)
        this.y = 0
      } else {
        this.x = randomRange(0, this.canvas.width)
        this.y = this.canvas.height
      }
    },
    startHorizontal() {
      if (this.xTarget < 0) {
        this.x = this.canvas.width
      } else {
        this.x = 0
      }
      this.y = this.canvas.height - this.bugHeight
    },
    initSpeedDirections() {
      if (this.isLowHorizontal || this.isNoisyLowHorizontal) {
        this.vx = Math.sign(this.xTarget) * this.currentSpeed
        this.vy = 0
        this.dx = this.vx
        this.dy = 0
      } else {
        let v = this.currentSpeed * Math.sqrt(2)
        if (this.y < this.canvas.height && this.y > 0) {
          this.vx = this.x <= 0 ? v : -v
          this.vy = plusOrMinus() * v
        } else {
          this.vx = plusOrMinus() * v
          this.vy = this.y <= 0 ? v : -v
        }
      }
    },
    initTargetDrift() {
      let targetOptions = this.targetDriftsOptions[this.bugsSettings.targetDrift]
      let xt = targetOptions.xTarget
      if (typeof xt === 'string') {
        xt = xt.split('+')
        xt = this.canvas[xt[0]] + Number(xt[1])
      }
      this.xTarget = xt
      this.yTarget = this.bugHeight
    },
    isInsideBoard() {
      return this.x > 0 && this.x < this.canvas.width && this.y > 0 && this.y < this.canvas.height
    },
    isHit(x, y) {
      return distance(x, y, this.x, this.y) <= this.currentBugSize / 1.5
    },
    escape(xe, ye) {
      let dist = distance(this.x, this.y, xe, ye)
      if (dist < this.canvas.height / 1.5) {
        this.dx = Math.abs(this.dx) * Math.sign(this.x - xe)
        this.dy = Math.abs(this.dy) * Math.sign(this.y - ye)
      }
    },
    rotate(dx, dy, angle) {
      return {
        dx: dx * Math.cos(angle) - dy * Math.sin(angle),
        dy: dx * Math.sin(angle) + dy * Math.cos(angle)
      }
    },
    getAngleRadians() {
      if (this.isMoveInCircles) {
        return Math.atan2(this.y - this.r0[1], this.x - this.r0[0]) + (this.bugsSettings.isAntiClockWise ? 0 : Math.PI)
      }
      return Math.atan2(this.dy, this.dx) + Math.PI / 2
    },
    collisionEffect(otherBug) {
      // collision between 2 bugs
      const angle = -Math.atan2(otherBug.y - this.y, otherBug.x - this.x)
      const u1 = this.rotate(this.dx, this.dy, angle)
      const u2 = this.rotate(otherBug.dx, otherBug.dy, angle)
      const v1 = {
        dx: ((this.mass - otherBug.mass) * u1.dx / (this.mass + otherBug.mass)) + (2 * otherBug.mass * u2.dx / (this.mass + otherBug.mass)),
        dy: u1.dy
      }
      const rotatedv1 = this.rotate(v1.dx, v1.dy, -angle)
      this.dx = rotatedv1.dx
      this.dy = rotatedv1.dy
      // const v2 = {
      //   dx: ((this.mass - otherBug.mass) * u2.dx / (this.mass + otherBug.mass)) + (2 * otherBug.mass * u1.dx / (this.mass + otherBug.mass)),
      //   dy: u2.dy
      // }
      // const rotatedv2 = this.rotate(v2.dx, v2.dy, -angle)
      // otherBug.dx = rotatedv2.dx
      // otherBug.dy = rotatedv2.dy
    }
  }
}
</script>
