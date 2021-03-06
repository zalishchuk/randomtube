<template>
	<div class="container">
		<section class="player">
			<div class="player__actions">
				<span class="actions__item">
					<b>/{{ currentBoard }}/</b>
				</span>
				<span class="actions__item actions__item--divide">|</span>
				<span class="actions__item">
					<a v-if="currentIndex != 0" href="#" @click="prev" @click.prevent.stop>Prev</a>
					<span v-else>Prev</span>
				</span>
				<span class="actions__item">
					<a href="#" @click="next" @click.prevent.stop>Next</a>
				</span>
				<span class="actions__item actions__item--divide">|</span>
				<span class="actions__item" :class="{ 'actions__item--active': options.loop }">
					<a href="#" @click="loop" @click.prevent.stop>Loop</a>
				</span>
				<span class="actions__item">
					<a v-if="currentVideo.url" :href="currentVideo.url" download>Download</a>
					<span v-else>Download</span>
				</span>
			</div>
			<video
				ref="video"
				class="player__video"

				:width="options.width"
				:height="options.height"
				:preload="options.preload"
				:controls="options.controls"
				:poster="options.poster"
				:loop="options.loop"
				:autoplay="options.autoplay"

				:src="currentVideo.url"
				:type="currentVideo.type">
				Your browser does not support the video tag.
			</video>
			<div class="player__name">
				<span v-if="currentVideo.name" v-text="currentVideo.name"></span>
				<small v-if="currentVideo.filesize">({{ currentVideo.filesize | formatBytes(2) }})</small>
			</div>
		</section>
		<!-- Some helpers -->
		<canvas ref="canvas" hidden></canvas>
		<span id="switchTheme" @click="switchTheme"></span>
	</div>
</template>

<script>
	import axios from 'axios'

	const instance = axios.create({
		baseURL: 'https://api.randomtube.xyz/',
		headers: {
			'Accept': 'application/json',
			'Content-Type': 'application/json',
		}
	})

	export default {
		name: 'player',
		data() {
			return {
				appTitle: 'randomTube',
				defaultHost: '2ch.hk',
				defaultBoard: 'b',

				currentChan: '2ch.hk',
				currentBoard: '',

				theme: 'white',

				$video: false,
				currentTitle: 'randomTube',
				currentPage: 1,
				currentIndex: 0,
				currentVideo: false,

				loading: false,
				playing: false,
				timer: null,

				options: {
					width: false,
					height: false,
					preload: true,
					controls: 'controls',
					poster: false,
					loop: false,
					autoplay: false
				},

				modal: false,
				$canvas: false,

				sources: []
			}
		},
		metaInfo() {
			return { title: this.currentTitle }
		},
		async mounted() {
			// Устанавливаем текущий раздел, тем самым в watch грузим видео и меняем заголовок
			this.currentBoard = this.$route.params.board || this.defaultBoard

			// Устанавливаем тему
			this.theme = localStorage.getItem('theme') || 'white'

			// Записываем экземлпяр видео плеера в общий Store
			this.$video = this.$refs.video
			this.$canvas = this.$refs.canvas

			// Перехватываем эвенты $video
			this.$video.addEventListener('click', e => {
				this.togglePlay()
			})
			this.$video.addEventListener('ended', e => {
				if (!this.options.loop) 
					this.timer = setTimeout(_ => {
						clearTimeout(this.timer)
						this.next()
					}, 1000)
			})
			this.$video.addEventListener('volumechange', e => { })
			this.$video.addEventListener('error', e => {
				this.sources.splice(this.currentIndex, 1)
				this.next()
			})
			// Перехватываем эвенты Window
			window.addEventListener('keydown', e => {
				switch(e.keyCode) {
					case 32: // Пробел
						this.togglePlay()
					break;
					case 37: // Лево
						this.prev()
					break;
					case 39: // Право
						this.next()
					break;
					case 70: // F key
						this.toggleFullscreen()
					break;
				}
			})
		},
		methods: {
			/**
			 * Основные действия плеера
			 */
			loop() {
				this.options.loop = !this.options.loop
			},
			prev() {
				// Запрещаем уходить в минус индексу
				if (this.currentIndex == 0) return false

				--this.currentIndex
			},
			next() {
				++this.currentIndex
			},
			togglePlay() {
				this.playing = !this.playing
				this.$video.paused ? this.$video.play() : this.$video.pause()
			},
			toggleFullscreen() {
				if (!document.fullscreenElement && // alternative standard method
					!document.mozFullScreenElement && !document.webkitFullscreenElement && !document.msFullscreenElement) {
					if (this.$video.requestFullscreen) {
						this.$video.requestFullscreen();
					} else if (this.$video.msRequestFullscreen) {
						this.$video.msRequestFullscreen();
					} else if (this.$video.mozRequestFullScreen) {
						this.$video.mozRequestFullScreen();
					} else if (this.$video.webkitRequestFullscreen) {
						this.$video.webkitRequestFullscreen();
					} else {
						console.log("Fullscreen API is not supported");
					}

				} else {

					if (document.exitFullscreen) {
						document.exitFullscreen();
					} else if (document.msExitFullscreen) {
						document.msExitFullscreen();
					} else if (document.mozCancelFullScreen) {
						document.mozCancelFullScreen();
					} else if (document.webkitCancelFullScreen) {
						document.webkitCancelFullScreen();
					}

				}
			},
			switchTheme() {
				this.theme = (this.theme == 'white') ? 'black' : 'white'
			},

			/**
			 * Офигенно важная функция по загрузке видео
			 */
			loadVideos(params) {
				return instance.get('video.get', { params })
				.then((data) => {
					const {response} = JSON.parse(data.request.response)
					response.items.forEach((video) => {
						this.sources.push(video)
					})
				})
				.catch((error) => {
					this.currentBoard = this.defaultBoard
				})
			},
			/**
			 * Поиск соуса.
			 * Не работает, ибо Абу пидорас ушастый.
			 */
			findSource() {
				// Pause video
				this.$video.pause()
				// Get context from canvas
				let context = this.$canvas.getContext('2d')
				// WTF ratio man
				let ratio = this.$video.videoWidth / this.$video.videoHeight,
				// Define the required width as 100 pixels smaller than the actual video's width
					w = this.$video.videoWidth - 100,
				// Calculate the height based on the video's width and the ratio
					h = parseInt(w / ratio, 10)
				// Set the canvas width and height to the values just calculated
				this.$canvas.width = w;
				this.$canvas.height = h;

				context.fillRect(0, 0, w, h);
				context.drawImage(this.$video, 0, 0, w, h);

				console.log(this.$canvas.toBlob())
			}
		},
		watch: {
			// При смене раздела
			async currentBoard() {
				// Грузим новые видео
				await this.loadVideos({ chan: this.currentChan, board: this.currentBoard }).then(_ => {
					this.currentVideo = this.sources[this.currentIndex]
				})
				// Меняем заголовок
				this.currentTitle = `${this.appTitle} - /${this.currentBoard}/`
				// Меняем путь
				this.$router.replace({ name: 'player-board', params: { board: this.currentBoard } })
			},
			currentIndex() {
				// Отчищаем таймер
				if (this.timer)
					clearTimeout(this.timer)

				this.currentVideo = this.sources[this.currentIndex]

				if (this.currentIndex == this.sources.length - 10)
					++this.currentPage

				this.$nextTick(_ => {
					this.$video.load()
					this.$video.play()
				})
			},
			async currentPage() {
				await this.loadVideos({ chan: this.currentChan, board: this.currentBoard, page: this.currentPage })
			},
			theme() {
				document.body.setAttribute('data-theme', this.theme)
				localStorage.setItem('theme', this.theme)
			}
		}
	}
</script>