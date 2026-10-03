window._plog = window._plog || function () { if (window.OPENTV_DEBUG) console.log.apply(console, arguments); };
window._pwarn = window._pwarn || function () { if (window.OPENTV_DEBUG) console.warn.apply(console, arguments); };
window._perr = window._perr || function () { if (window.OPENTV_DEBUG) console.error.apply(console, arguments); };

class VideoPlayer {
    constructor() {
        this.hls = null;
        this.mpegtsPlayer = null;
        this.container = document.getElementById('player-modal');
        this.videoEl = document.getElementById('video-player');
        this.titleEl = document.getElementById('player-title');
        this.loader = document.getElementById('player-loader');
        this.errorBox = document.getElementById('player-error');
        this.errorText = document.getElementById('player-error-text');
        this.retryBtn = document.getElementById('btn-retry');
        this.controls = document.getElementById('player-controls');
        this.btnPlayPause = document.getElementById('btn-play-pause');
        this.btnFullscreen = document.getElementById('btn-fullscreen');
        this.btnMute = document.getElementById('btn-mute');
        this.volumeSlider = document.getElementById('volume-slider');
        this.timeDisplay = document.getElementById('player-time');
        this.liveBadge = document.getElementById('live-badge');
        this.currentUrl = '';
        this.currentTitle = '';
        this.currentType = '';
        this.isLive = false;
        this.hideTimer = null;
        this.retryCount = 0;
        this.maxRetries = 8;
        this.playAttempt = 0;
        this.mediaRecoverCount = 0;
        this._mkvMpegtsTried = false;
        this._cameFromHls = false;
        this._vodMpegtsTried = false;
        this._inMkvFallback = false;
        this._mkvMpegtsTimer = null;

        this.seekBar = document.getElementById('seek-bar');
        this.seekBarWrapper = document.getElementById('seek-bar-wrapper');
        this.seekBuffered = document.getElementById('seek-buffered');
        this.seekProgress = document.getElementById('seek-progress');
        this.seekThumb = document.getElementById('seek-thumb');
        this.seekTooltip = document.getElementById('seek-tooltip');
        this.seekGroup = document.getElementById('seek-group');
        this.timeCurrent = document.getElementById('player-time-current');
        this.timeDuration = document.getElementById('player-time-duration');
        this.btnBackward = document.getElementById('btn-backward');
        this.btnForward = document.getElementById('btn-forward');
        this.btnPlayCenter = document.getElementById('btn-play-center');
        this.btnSubtitles = document.getElementById('btn-subtitles');
        this.btnAudioTrack = document.getElementById('btn-audio-track');
        this.btnSettingsPlayer = document.getElementById('btn-settings-player');
        this.settingsPanel = document.getElementById('player-settings-panel');
        this.subtitleOptions = document.getElementById('subtitle-options');
        this.audioOptions = document.getElementById('audio-options');
        this.qualityOptions = document.getElementById('quality-options');
        this.centerControls = document.getElementById('player-center-controls');

        this.seekDragging = false;

        this.currentStreamId = null;
        this.resumeAt = 0;
        this.progressSaveTimer = null;
        this.subtitleSize = 1;
        this.subtitleStyle = 0;
        this.subtitleDelay = 0;
        this.loadSubtitlePrefs();

        this.closeBtn = document.getElementById('btn-close-player');
        this.closeBtn?.addEventListener('click', () => this.stop());
        this.retryBtn?.addEventListener('click', () => this.retry());
        this.btnPlayPause?.addEventListener('click', () => this.togglePlay());
        this.btnFullscreen?.addEventListener('click', () => this.toggleFullscreen());
        document.addEventListener('fullscreenchange', () => this._updateFsIcon());
        document.addEventListener('webkitfullscreenchange', () => this._updateFsIcon());
        this.btnMute?.addEventListener('click', () => this.toggleMute());
        this.volumeSlider?.addEventListener('input', (e) => this.setVolume(e.target.value));
        this.videoEl?.addEventListener('click', () => this.togglePlay());
        this.videoEl?.addEventListener('dblclick', () => this.toggleFullscreen());
        this.videoEl?.addEventListener('timeupdate', () => this.updateTime());
        this.videoEl?.addEventListener('ended', () => this.saveProgressNow());
        this.videoEl?.addEventListener('playing', () => this._reportLive(true));
        this.videoEl?.addEventListener('ended', () => this._onLiveEnded());

        // Watchdog de congelamento (live): o proxy corta streams longos (~60s); se o video
        // parar de avancar por ~9s sem ser pause/seek, reconecta invisivelmente
        this._stallLastTime = 0;
        this._stallStrikes = 0;
        setInterval(() => {
            if (!this.isLive || !this.currentUrl || !this.videoEl) return;
            if (this._reconnectTimer) { this._stallStrikes = 0; return; } // reconexao ja agendada — deixa o timer cuidar
            if (this.videoEl.paused || this.videoEl.seeking) { this._stallStrikes = 0; return; }
            const ct = this.videoEl.currentTime;
            if (ct < this._stallLastTime - 1) { this._stallLastTime = ct; this._stallStrikes = 0; return; } // tempo voltou (nova MediaSource pos-restart)
            if (ct > this._stallLastTime + 0.2) { this._stallLastTime = ct; this._stallStrikes = 0; return; }
            this._stallStrikes++;
            if (this._stallStrikes >= 3) {
                this._stallStrikes = 0;
                _plog('[LIVE] Stall detectado — reconectando stream');
                this._restartLive();
            }
        }, 3000);

        document.getElementById('btn-resume-continue')?.addEventListener('click', () => {
            const pos = this._pendingResume || 0;
            this.hideResumeModal();
            this.resumeAt = pos;
            if (this.videoEl && pos > 0) {
                const trySeek = () => {
                    if (isFinite(this.videoEl.duration) && this.videoEl.duration > pos) {
                        this.videoEl.currentTime = pos;
                    } else if (this.videoEl.readyState >= 1) {
                        setTimeout(trySeek, 300);
                    }
                };
                trySeek();
            }
            this.videoEl?.play().catch(() => {});
        });
        document.getElementById('btn-resume-restart')?.addEventListener('click', () => {
            this.hideResumeModal();
            this.resumeAt = 0;
            this._pendingResume = 0;
            if (this.videoEl) this.videoEl.currentTime = 0;
            this.videoEl?.play().catch(() => {});
        });

        this.btnPlayCenter?.addEventListener('click', (e) => { e.stopPropagation(); this.togglePlay(); });

        this.btnBackward?.addEventListener('click', () => this.skip(-10));
        this.btnForward?.addEventListener('click', () => this.skip(10));

        if (this.seekBarWrapper) {
            this.seekBarWrapper.addEventListener('mousedown', (e) => this.onSeekStart(e));
            this.seekBarWrapper.addEventListener('mousemove', (e) => this.onSeekHover(e));
            this.seekBarWrapper.addEventListener('mouseleave', () => this.onSeekHoverEnd());
        }
        document.addEventListener('mousemove', (e) => this.onSeekDragMove(e));
        document.addEventListener('mouseup', (e) => this.onSeekDragEnd(e));

        this.btnSettingsPlayer?.addEventListener('click', (e) => { e.stopPropagation(); this.toggleSettings(); });
        document.addEventListener('click', (e) => {
            if (this.settingsPanel && !this.settingsPanel.classList.contains('hidden')) {
                if (this.settingsPanel.contains(e.target)) return;
                if (this.btnSettingsPlayer && this.btnSettingsPlayer.contains(e.target)) return;
                this.settingsPanel.classList.add('hidden');
            }
        });

        this.btnSubtitles?.addEventListener('click', () => this.toggleSettings());
        this.btnAudioTrack?.addEventListener('click', () => this.toggleSettings());

        document.querySelectorAll('#subtitle-size-options .settings-option').forEach(el => {
            el.addEventListener('click', () => {
                this.subtitleSize = parseInt(el.dataset.subSize, 10);
                document.querySelectorAll('#subtitle-size-options .settings-option').forEach(o => o.classList.remove('active'));
                el.classList.add('active');
                this.applySubtitleStyle();
                this.saveSubtitlePrefs();
            });
        });
        document.querySelectorAll('#subtitle-style-options .settings-option').forEach(el => {
            el.addEventListener('click', () => {
                this.subtitleStyle = parseInt(el.dataset.subStyle, 10);
                document.querySelectorAll('#subtitle-style-options .settings-option').forEach(o => o.classList.remove('active'));
                el.classList.add('active');
                this.applySubtitleStyle();
                this.saveSubtitlePrefs();
            });
        });
        document.getElementById('btn-sub-delay-minus')?.addEventListener('click', () => this.adjustSubtitleDelay(-0.5));
        document.getElementById('btn-sub-delay-plus')?.addEventListener('click', () => this.adjustSubtitleDelay(0.5));
        this.updateSubDelayLabel();

        document.addEventListener('keydown', (e) => {
            if (!this.container || this.container.classList.contains('hidden')) return;
            if (e.key === 'Escape') {
                if (this.settingsPanel && !this.settingsPanel.classList.contains('hidden')) {
                    this.settingsPanel.classList.add('hidden');
                    return;
                }
                this.stop();
            }
            if (e.key === ' ') { e.preventDefault(); this.togglePlay(); }
            if (e.key === 'f' || e.key === 'F') this.toggleFullscreen();
            if (e.key === 'ArrowRight') this.skip(10);
            if (e.key === 'ArrowLeft') this.skip(-10);
        });
    }

    play(streamUrl, title, type = 'live', opts = {}) {
        this.cleanup();
        this.currentUrl = streamUrl;
        this.currentTitle = title;
        this.currentType = type;
        this.isLive = type === 'live';
        this.retryCount = 0;
        this.playAttempt = 0;
        this.mediaRecoverCount = 0;
        this._mkvMpegtsTried = false;
        this._cameFromHls = false;
        this._vodMpegtsTried = false;
        this._inMkvFallback = false;
        if (this._mkvMpegtsTimer) { clearTimeout(this._mkvMpegtsTimer); this._mkvMpegtsTimer = null; }
        this.currentStreamId = opts.streamId != null ? opts.streamId : null;
        this.currentVariants = Array.isArray(opts.variants) ? opts.variants : [];
        this._liveStatusSent = false;
        this._restartCount = 0;
        this._lastLiveEngine = null;
        this._liveBackoff = 2;
        clearTimeout(this._reconnectTimer);
        this.resumeAt = 0;
        this._pendingResume = opts.resumeAt > 0 && !this.isLive ? opts.resumeAt : 0;
        this.titleEl.textContent = title;
        this.container.classList.remove('hidden');
        document.body.style.overflow = 'hidden';
        if (this.liveBadge) this.liveBadge.style.display = this.isLive ? 'inline-block' : 'none';
        this.errorBox.classList.add('hidden');
        this.loader.classList.remove('hidden');
        this.updateSeekUI();
        this.updateCenterPlayBtn();
        this.hideSettingsPanel();
        this.applySubtitleStyle();
        this.startProgressSaver();
        if (!this.isLive && this._pendingResume > 5) {
            this.showResumeModal(this._pendingResume);
        }
        this.startStream(streamUrl);
    }

    loadSubtitlePrefs() {
        try {
            const p = JSON.parse(localStorage.getItem('unitv_subtitle_settings') || '{}');
            if (typeof p.size === 'number') this.subtitleSize = p.size;
            if (typeof p.style === 'number') this.subtitleStyle = p.style;
            if (typeof p.delay === 'number') this.subtitleDelay = p.delay;
        } catch (e) {}
        const sizeOpts = document.querySelectorAll('#subtitle-size-options .settings-option');
        sizeOpts.forEach(o => o.classList.toggle('active', parseFloat(o.dataset.subSize) === this.subtitleSize));
        const styleOpts = document.querySelectorAll('#subtitle-style-options .settings-option');
        styleOpts.forEach(o => o.classList.toggle('active', parseInt(o.dataset.subStyle, 10) === this.subtitleStyle));
        this.updateSubDelayLabel();
        this.applySubtitleStyle();
    }

    applySubtitlePrefs() {
        this.loadSubtitlePrefs();
        if (this._maybeLoadSubs) this._maybeLoadSubs();
    }

    _maybeLoadSubs() {
        if (!this.videoEl || this.isLive) return;
        if (this.currentType !== 'movie' && this.currentType !== 'series' && this.currentType !== 'vod') return;
        if (this.currentStreamId == null || !window.SubtitleStore) return;
        const id = this.currentStreamId;
        SubtitleStore.loadForVod(id, this.videoEl).then((list) => {
            if (this.currentStreamId !== id) return;
            this.populateNativeSubtitles();
            if (!list || !list.length) {
                const box = this.subtitleOptions;
                if (box && box.querySelector('.settings-option-text')) {
                    box.insertAdjacentHTML('beforeend',
                        '<div class="settings-option" id="sub-local-opt"><span class="settings-option-text">Carregar legenda local</span></div>');
                    document.getElementById('sub-local-opt')?.addEventListener('click', () => {
                        SubtitleStore.createLocalInput(this.videoEl);
                    });
                }
            }
        }).catch(() => {});
    }

    saveSubtitlePrefs() {
        try { localStorage.setItem('unitv_subtitle_settings', JSON.stringify({ size: this.subtitleSize, style: this.subtitleStyle, delay: this.subtitleDelay })); } catch (e) {}
    }

    adjustSubtitleDelay(delta) {
        this.subtitleDelay = Math.round((this.subtitleDelay + delta) * 10) / 10;
        this.subtitleDelay = Math.max(-10, Math.min(10, this.subtitleDelay));
        this.updateSubDelayLabel();
        this.saveSubtitlePrefs();
        this.applySubtitleDelay();
    }

    updateSubDelayLabel() {
        const el = document.getElementById('sub-delay-value');
        if (el) el.textContent = (this.subtitleDelay > 0 ? '+' : '') + this.subtitleDelay.toFixed(1) + 's';
    }

    applySubtitleStyle() {
        let styleEl = document.getElementById('cue-style');
        if (!styleEl) {
            styleEl = document.createElement('style');
            styleEl.id = 'cue-style';
            document.head.appendChild(styleEl);
        }
        const sizeMap = { 0: '0.8em', 1: '1em', 2: '1.4em' };
        const styles = [
            { color: 'rgba(255,255,255,1)', bg: 'rgba(0,0,0,0.75)' },
            { color: 'rgb(255,220,0)', bg: 'rgba(0,0,0,0.75)' },
            { color: 'rgba(255,255,255,1)', bg: 'rgba(0,0,0,0)' }
        ];
        const s = styles[this.subtitleStyle] || styles[0];
        const fs = sizeMap[this.subtitleSize] || '1em';
        styleEl.textContent = '#video-player::cue{font-size:' + fs + ';color:' + s.color + ';background:' + s.bg + ';}';
    }

    applySubtitleDelay() {
        if (this.hls && this.hls.subtitleTrack >= 0) {
            try { this.hls.subtitleDisplay = false; setTimeout(() => { if (this.hls) this.hls.subtitleDisplay = true; }, 50); } catch (e) {}
        }
        const tracks = this.videoEl?.textTracks;
        if (tracks) {
            for (let i = 0; i < tracks.length; i++) {
                if (tracks[i].mode === 'showing') {
                    try { tracks[i].mode = 'hidden'; setTimeout(() => { if (tracks[i]) tracks[i].mode = 'showing'; }, 50); } catch (e) {}
                }
            }
        }
    }

    startProgressSaver() {
        this.stopProgressSaver();
        if (this.isLive || this.currentStreamId == null) return;
        this.progressSaveTimer = setInterval(() => this.saveProgressNow(), 5000);
    }

    stopProgressSaver() {
        if (this.progressSaveTimer) { clearInterval(this.progressSaveTimer); this.progressSaveTimer = null; }
    }

    saveProgressNow() {
        if (this.isLive || this.currentStreamId == null || !this.videoEl) return;
        if (!isFinite(this.videoEl.duration) || this.videoEl.duration <= 0) return;
        if (typeof WatchStore !== 'undefined') {
            WatchStore.saveProgress(this.currentType, this.currentStreamId, this.videoEl.currentTime, this.videoEl.duration, this.currentTitle);
        }
    }

    resumeIfPossible() {
        if (this.resumeAt > 0 && !this.isLive && isFinite(this.videoEl.duration) && this.videoEl.duration > this.resumeAt) {
            const target = this.resumeAt;
            this.resumeAt = 0;
            const trySeek = () => {
                if (this.videoEl.readyState >= 1) {
                    this.videoEl.currentTime = target;
                } else {
                    setTimeout(trySeek, 200);
                }
            };
            trySeek();
        } else if (this.resumeAt > 0 && !this.isLive && (!isFinite(this.videoEl.duration) || this.videoEl.duration <= 0)) {
            const onMeta = () => {
                if (this.resumeAt > 0 && isFinite(this.videoEl.duration) && this.videoEl.duration > this.resumeAt) {
                    this.videoEl.currentTime = this.resumeAt;
                    this.resumeAt = 0;
                }
                this.videoEl.removeEventListener('loadedmetadata', onMeta);
            };
            this.videoEl.addEventListener('loadedmetadata', onMeta);
        }
    }

    maybeSeekResume() {
        if (this.resumeAt > 0 && !this.isLive) this.resumeIfPossible();
    }

    showResumeModal(position) {
        const overlay = document.getElementById('resume-modal');
        if (!overlay) return;
        const label = document.getElementById('resume-position');
        if (label) label.textContent = this.fmt(position);
        this._pendingResume = position;
        overlay.classList.remove('hidden');
        try { this.videoEl?.pause(); } catch (e) {}
        document.getElementById('btn-resume-continue')?.focus();
    }

    hideResumeModal() {
        document.getElementById('resume-modal')?.classList.add('hidden');
        this._pendingResume = 0;
    }

    startStream(url) {
        this.loader.classList.remove('hidden');
        this.errorBox.classList.add('hidden');

        const isVOD = this.currentType === 'movie' || this.currentType === 'series' || this.currentType === 'vod';
        const isMP4 = url.match(/\.(mp4|mkv|avi|webm)(\?|$)/i);

        if (isVOD) {
            this.playVideo(url);
            return;
        }

        if (isMP4) {
            this.playDirect(url);
            return;
        }

        this.playAttempt++;
        const hlsUrl = this.toHlsUrl(url);
        const viaProxy = typeof window !== 'undefined' && url.indexOf(window.location.origin + '/') === 0;

        if (viaProxy && typeof Hls !== 'undefined' && Hls.isSupported()) {
            this.playHLS(hlsUrl);
        } else if (typeof mpegts !== 'undefined' && mpegts.isSupported() && this.playAttempt <= 2) {
            this.playMpegts(url);
        } else if (typeof Hls !== 'undefined' && Hls.isSupported()) {
            this.playHLS(hlsUrl);
        } else if (this.videoEl.canPlayType('application/vnd.apple.mpegurl')) {
            this.playNativeHLS(hlsUrl);
        } else {
            this.playDirect(url);
        }
    }

    playMpegts(url) {
        this.loader.classList.remove('hidden');
        _plog('[MPEGTS] Starting live stream');
        try {
            this.destroyMpegts();

            this.mpegtsPlayer = mpegts.createPlayer({
                type: 'mpegts',
                url: url,
                isLive: this.isLive,
                cors: true,
                hasAudio: true,
                hasVideo: true
            }, {
                enableWorker: false,
                enableStashBuffer: true,
                stashInitialSize: 384,
                autoCleanupSourceBuffer: true,
                autoCleanupMaxBackwardDuration: 30,
                autoCleanupMinBackwardDuration: 10,
                liveBufferLatencyChasing: false
            });

            if (this.isLive) this._lastLiveEngine = 'mpegts';

            // Proxy edge corta streams longos (~60s). Quando a conexao morre (LOADING_COMPLETE),
            // agenda a reconexao pra ANTES do buffer acabar — com BACKOFF exponencial pra nao
            // martelar o CDN (throttle: pulls repetidos em sequencia sao cortados ainda mais curtos).
            this._liveConnStart = Date.now();
            this.mpegtsPlayer.on(mpegts.Events.LOADING_COMPLETE, () => {
                if (!this.isLive || !this.currentUrl) return;
                const dur = (Date.now() - (this._liveConnStart || Date.now())) / 1000;
                if (dur < 15) this._liveBackoff = Math.min((this._liveBackoff || 2) * 2, 30);
                else if (dur > 30) this._liveBackoff = 2;
                let ahead = 0;
                try {
                    const b = this.videoEl.buffered;
                    if (b.length) ahead = Math.max(0, b.end(b.length - 1) - this.videoEl.currentTime);
                } catch (e) {}
                const minWait = (this._liveBackoff || 2) * 1000;
                // Buffer apertado (<=6s): volta ja — esperar backoff com pouco buffer = tela travada.
                // Com folga: espera porem o backoff e limitado a 4s (provedor corta conexoes em
                // segundos de forma normal; piso longo = underrun garantido).
                let wait;
                if (ahead <= 6) wait = 300;
                else wait = Math.max(Math.min(minWait, 4000), Math.min((ahead - 3) * 1000, 12000));
                _plog('[LIVE] Conexao durou ' + dur.toFixed(0) + 's (buffer: ' + ahead.toFixed(1) + 's) — reconexao em ' + Math.round(wait / 1000) + 's' + (minWait > 2000 ? ' (backoff)' : '') + ')');
                clearTimeout(this._reconnectTimer);
                this._reconnectTimer = setTimeout(() => this._restartLive(), wait);
            });

            this.mpegtsPlayer.attachMediaElement(this.videoEl);
            this.mpegtsPlayer.load();
this.mpegtsPlayer.play();

            // Preenche seletor de qualidade/provedor se houver variantes
            if (this.isLive && this.currentVariants && this.currentVariants.length) {
                this.populateStreamVariants();
            }

            let hasPlayed = false;
            const onPlaying = () => {
                hasPlayed = true;
                this.loader.classList.add('hidden');
                this.videoEl.removeEventListener('playing', onPlaying);
            };
            this.videoEl.addEventListener('playing', onPlaying);

            setTimeout(() => {
                if (!hasPlayed && !this.loader.classList.contains('hidden')) {
                    this.loader.classList.add('hidden');
                }
            }, 8000);

            this.mpegtsPlayer.on(mpegts.Events.ERROR, (errType, errDetail, errInfo) => {
                _pwarn('[MPEGTS]', errType, errDetail);
                const msg = JSON.stringify(errInfo || {});
                const hevcUnsupported = /hvc1|hev1|MediaMSEError|addSourceBuffer/i.test(msg + errDetail) &&
                    !this._mpegtsHevcNotified;
if (hevcUnsupported && /hvc1|hev1|MediaMSEError/i.test(msg + errDetail)) {
                    this._mpegtsHevcNotified = true;
                    this.destroyMpegts();
                    // tenta trocar automaticamente para variante H264 (igual HLS)
                    this._hevcFallback();
                    return;
                }
                if (this._inMkvFallback) {
                    this.destroyMpegts();
                    this.showMkvError();
                    return;
                }
                if (errType === mpegts.ErrorTypes.NETWORK_ERROR || errType === mpegts.ErrorTypes.MEDIA_ERROR) {
                    _plog('[MPEGTS] Error, trying fallback playback');
                    this.destroyMpegts();
                    if (this.isLive && !this._cameFromHls) {
                        this.playHLS(this.toHlsUrl(url));
                    } else {
                        this.playDirect(url);
                    }
                }
            });

        } catch (e) {
            _pwarn('[MPEGTS] Failed:', e.message);
            if (this._inMkvFallback) {
                this.showMkvError();
            } else {
                _plog('[MPEGTS] Trying direct fallback');
                if (this.isLive) this.playHLS(this.toHlsUrl(url));
                else this.playDirect(url);
            }
        }
    }

    playHLS(url) {
        this.loader.classList.remove('hidden');
        if (this.hls) { this.hls.destroy(); this.hls = null; }
        if (this.isLive) this._lastLiveEngine = 'hls';

        this.hls = new Hls({
            enableWorker: true,
            lowLatencyMode: false,
            maxBufferLength: this.isLive ? 26 : 30,
            maxMaxBufferLength: this.isLive ? 60 : 120,
            backBufferLength: 30,
            startFragPrefetch: false,
            maxBufferHole: 0.8,
            highBufferWatchdogPeriod: 2,
            nudgeOffset: 0.2,
            nudgeMaxRetry: 5,
            maxFragLookUpTolerance: 0.25,
            abrEwmaDefaultEstimate: 500000,
            testBandwidth: false,
            progressive: false,
            manifestLoadPolicy: {
                default: {
                    maxTimeToFirstByteMs: 8000,
                    maxLoadTimeMs: 15000,
                    timeoutRetry: { maxNumRetry: 3, retryDelayMs: 500, maxRetryDelayMs: 4000 },
                    errorRetry: { maxNumRetry: 3, retryDelayMs: 800, maxRetryDelayMs: 5000 }
                }
            },
            playlistLoadPolicy: {
                default: {
                    maxTimeToFirstByteMs: 8000,
                    maxLoadTimeMs: 15000,
                    timeoutRetry: { maxNumRetry: 3, retryDelayMs: 500, maxRetryDelayMs: 4000 },
                    errorRetry: { maxNumRetry: 3, retryDelayMs: 800, maxRetryDelayMs: 5000 }
                }
            },
            fragLoadPolicy: {
                default: {
                    maxTimeToFirstByteMs: 8000,
                    maxLoadTimeMs: 60000,
                    timeoutRetry: { maxNumRetry: 5, retryDelayMs: 500, maxRetryDelayMs: 4000 },
                    errorRetry: { maxNumRetry: 8, retryDelayMs: 800, maxRetryDelayMs: 8000 }
                }
            }
        });

        this.hls.loadSource(url);
        this.hls.attachMedia(this.videoEl);

        this.hls.on(Hls.Events.MANIFEST_PARSED, () => {
            this.retryCount = 0;
            this.loader.classList.add('hidden');
            this.smartPlay();
            this.populateHLSSubtitles();
            this.populateHLSAudioTracks();
            this.populateHLSQualityInfo();
        });

        this.hls.on(Hls.Events.FRAG_BUFFERED, () => {
            this.loader.classList.add('hidden');
            try { if (this.hls && this.hls.bandwidthEstimate) this.updateBandwidthDisplay(this.hls.bandwidthEstimate); } catch (e) {}
        });

        this.hls.on(Hls.Events.SUBTITLE_TRACKS_UPDATED, () => {
            this.populateHLSSubtitles();
        });

        this.hls.on(Hls.Events.AUDIO_TRACKS_UPDATED, () => {
            this.populateHLSAudioTracks();
        });

        this.hls.on(Hls.Events.ERROR, (_, data) => {
            _pwarn('[HLS]', data.type, data.details, data.fatal);
            if (data.fatal) {
                switch (data.type) {
                    case Hls.ErrorTypes.NETWORK_ERROR:
                        // .m3u8 que na verdade e TS cru OU corpo invalido pontual do provedor:
                        // em vez de cair direto pro mpegts, tenta de novo via startLoad (rede recuperavel)
                        if (data.details === 'levelParsingError') {
                            if (this.retryCount < this.maxRetries) {
                                this.retryCount++;
                                _plog('[HLS] Manifest invalido (corpo upstream) — retry ' + this.retryCount + '/' + this.maxRetries);
                                const delay = 1200 * this.retryCount;
                                setTimeout(() => {
                                    try { if (this.hls) this.hls.startLoad(); } catch (e) {}
                                }, delay);
                                return;
                            }
                            _plog('[HLS] Manifest invalido apos retries — caindo pro mpegts');
                            this.hls.destroy(); this.hls = null;
                            this._hlsToFallback(url);
                            return;
                        }
                        // 404/403 no manifest: canal sem variante HLS (ou offline) — sem retry
                        if (data.details === 'manifestLoadError' && data.response && (data.response.code === 404 || data.response.code === 403)) {
                            _plog('[HLS] Manifest 404 — canal sem HLS, indo pro mpegts');
                            this.hls.destroy(); this.hls = null;
                            this._hlsToFallback(url);
                            return;
                        }
                        if (this.retryCount < this.maxRetries) {
                            this.retryCount++;
                            setTimeout(() => {
                                try { this.hls.startLoad(); } catch(e) {}
                            }, 1000 * this.retryCount);
                        } else {
                            _plog('[HLS] Network errors exhausted, trying fallback');
                            this.hls.destroy(); this.hls = null;
                            this._hlsToFallback(url);
                        }
                        break;
                    case Hls.ErrorTypes.MEDIA_ERROR:
                        if (data.details === 'bufferAddCodecError') {
                            const codecInfo = (data.error && data.error.codec) || '';
                            const hevc = /hvc|hev/i.test(codecInfo) || /hvc|hev/i.test(JSON.stringify(data));
                            _plog('[HLS] Codec not supported by MSE:', codecInfo || 'unknown');
                            this.hls.destroy(); this.hls = null;
                            if (hevc) {
                                this.showErrorMessage('Canal em HEVC (H.265). Procurando versão compatível...');
                                this._hevcFallback();
                            } else {
                                this._hlsToFallback(url);
                            }
                            return;
                        }
                        if (this.mediaRecoverCount < 2) {
                            this.mediaRecoverCount++;
                            try { this.hls.recoverMediaError(); } catch (e) {
                                _plog('[HLS] Recovery failed, trying fallback');
                                this.hls.destroy(); this.hls = null;
                                this._hlsToFallback(url);
                            }
                        } else {
                            _plog('[HLS] Media errors exhausted, trying fallback');
                            this.hls.destroy(); this.hls = null;
                            this._hlsToFallback(url);
                        }
                        break;
                    default:
                        _plog('[HLS] Fatal error, trying fallback');
                        this.hls.destroy(); this.hls = null;
                        this._hlsToFallback(url);
                        break;
                }
            }
        });
    }

    // Fallback pos-HLS: canal ao vivo (TS cru) â†’ mpegts.js; conteudo VOD â†’ direct
    _hlsToFallback(hlsUrl) {
        const tsUrl = String(hlsUrl || '').replace(/\.m3u8(?=\?|$)/i, '.ts');
        const mpegtsOk = typeof mpegts !== 'undefined' && mpegts.isSupported();
        if (this.isLive && mpegtsOk) {
            this._cameFromHls = true; // evita voltar pro HLS se o mpegts falhar (ping-pong)
            this.playMpegts(tsUrl || hlsUrl);
            return;
        }
        this.playDirect(tsUrl || hlsUrl);
    }

    async probeVodFormat(url) {
        try {
            const res = await fetch(url, {
                headers: { Range: 'bytes=0-15' },
                signal: AbortSignal.timeout ? AbortSignal.timeout(6000) : undefined
            });
            const buf = new Uint8Array(await res.arrayBuffer());
            const ctype = (res.headers.get('content-type') || '').toLowerCase();
            const acao = res.headers.get('access-control-allow-origin');
            const mkv = (buf[0] === 0x1A && buf[1] === 0x45 && buf[2] === 0xDF && buf[3] === 0xA3) || ctype.includes('matroska') || ctype.includes('mkv');
            const ts = buf[0] === 0x47 || ctype.includes('mp2t') || ctype.includes('mpegts');
            const mp4 = (buf.length >= 8 && buf[4] === 0x66 && buf[5] === 0x74) || ctype.includes('mp4');
            _plog('[PROBE]', { status: res.status, ctype, acao, mkv, ts, mp4 });
            if (mkv) return 'mkv';
            if (ts) return 'ts';
            if (mp4) return 'mp4';
            if (acao) return 'unknown-cors';
            return 'unknown-nocors';
        } catch (e) {
            _pwarn('[PROBE] failed:', e.message);
            return 'unknown';
        }
    }

    async playVideo(url) {
        this.loader.classList.remove('hidden');
        _plog('[VIDEO] VOD:', url.substring(0, 100));

        if (this.mseFallbackTimer) { clearTimeout(this.mseFallbackTimer); this.mseFallbackTimer = null; }
        if (this.mseAbortController) { this.mseAbortController.abort(); this.mseAbortController = null; }

        this._vodProbe = null;
        this.probeVodFormat(url).then(f => {
            if (this.currentUrl === url) this._vodProbe = f;
        }).catch(() => {});

        this.videoEl.src = url;
        this.videoEl.load();

        let resolved = false;
        const finish = () => {
            if (resolved) return;
            resolved = true;
            if (this.mseFallbackTimer) { clearTimeout(this.mseFallbackTimer); this.mseFallbackTimer = null; }
            this.loader.classList.add('hidden');
            this.smartPlay();
            this._maybeLoadSubs();
            this.populateNativeSubtitles();
            this.populateNativeQualityInfo();
        };

        this.videoEl.oncanplay = finish;
        this.videoEl.onloadedmetadata = () => { finish(); this.resumeIfPossible(); };
        this.videoEl.onloadeddata = () => { finish(); this.resumeIfPossible(); };

        this.mseFallbackTimer = setTimeout(() => {
            if (!resolved && this.videoEl.readyState < 2) {
                _plog('[VIDEO] Native stuck rs=' + this.videoEl.readyState + ', aborting...');
                this.videoEl.onloadeddata = null;
                this.videoEl.oncanplay = null;
                this.videoEl.onloadedmetadata = null;
                this.videoEl.onerror = null;
                this.videoEl.removeAttribute('src');
                this.videoEl.load();
                this.recoverVod(url);
            }
        }, 25000);

        this.videoEl.onerror = () => {
            if (resolved) return;
            if (this.mseFallbackTimer) { clearTimeout(this.mseFallbackTimer); this.mseFallbackTimer = null; }
            const err = this.videoEl.error;
            _pwarn('[VIDEO] Error:', err?.code, err?.message);
            this.videoEl.onerror = null;
            this.videoEl.removeAttribute('src');
            this.videoEl.load();
            this.recoverVod(url);
        };
    }

    async recoverVod(url) {
        if (this.currentUrl !== url) return;
        let fmt = this._vodProbe;
        if (!fmt) fmt = await this.probeVodFormat(url);
        if (this.currentUrl !== url) return;
        _plog('[VIDEO] recover fmt=', fmt);

        if (fmt === 'mp4' || fmt === 'unknown-cors') {
            this.playVideoMSE(url);
            return;
        }

const mpegtsOk = typeof mpegts !== 'undefined' && mpegts.isSupported();
        _plog('[VIDEO] recover mpegtsOk=', mpegtsOk, 'tried=', this._vodMpegtsTried);
        if ((fmt === 'ts' || fmt === 'unknown' || fmt === 'unknown-nocors' || fmt === 'unknown-cors') && mpegtsOk && !this._vodMpegtsTried) {
            this._vodMpegtsTried = true;
            _plog('[VIDEO] Trying mpegts for', fmt);
            this.playMpegts(url);
            const tsTimer = setTimeout(() => {
                if (this.videoEl && this.videoEl.readyState < 2 && this.currentUrl === url) {
                    this.destroyMpegts();
                    if (fmt === 'ts' || fmt === 'unknown-nocors') {
                        // Erro mais específico para 403/401 (token expirado)
                        this.showErrorMessage('Falha ao carregar stream (403/401). Tente novamente ou verifique se a conta tem acesso a este conteúdo.');
                    } else {
                        this.playVideoMSE(url);
                    }
                }
            }, 10000);
            this.videoEl.addEventListener('loadedmetadata', () => clearTimeout(tsTimer), { once: true });
            this.videoEl.addEventListener('playing', () => clearTimeout(tsTimer), { once: true });
            return;
        }

        if (fmt === 'mkv' || this.isMkvUrl(url)) {
            this.tryMkvFallback(url);
            return;
        }

        this.playVideoMSE(url);
    }

    playVideoMSE(url) {
        if (this.mseAbortController) { this.mseAbortController.abort(); this.mseAbortController = null; }
        if (this._mseEvictionTimer) { clearInterval(this._mseEvictionTimer); this._mseEvictionTimer = null; }
        this.loader.classList.remove('hidden');
        _plog('[MSE] Starting MP4 playback via MSE:', url.substring(0, 100));

        if (!window.MediaSource || !window.MP4Box) {
            _pwarn('[MSE] MediaSource or MP4Box not available, falling back');
            this.playDirect(url);
            return;
        }

        let mp4boxFile = null;
        let mediaSource = null;
        let videoSB = null;
        let audioSB = null;
        let fileOffset = 0;
        let aborted = false;
        let videoAppending = false;
        let audioAppending = false;
        let videoInitSegment = null;
        let audioInitSegment = null;
        let videoInitAppended = false;
        let audioInitAppended = false;
        let videoSegQueue = [];
        let audioSegQueue = [];
        let evictionTimer = null;
        let evicting = false;
        let pendingEviction = false;

        const cleanup = () => {
            aborted = true;
            pendingEviction = false;
            if (evictionTimer) { clearInterval(evictionTimer); evictionTimer = null; }
            if (this._mseWatchdog) { clearInterval(this._mseWatchdog); this._mseWatchdog = null; }
            if (mp4boxFile) { try { mp4boxFile.flush(); } catch(e) {} mp4boxFile = null; }
            if (this.mseAbortController) { this.mseAbortController.abort(); this.mseAbortController = null; }
        };

        const evict = (sb, aggressive) => {
            if (!sb || sb.updating) return false;
            try {
                const b = sb.buffered;
                const ct = this.videoEl.currentTime;
                if (b.length === 0) return false;
                for (let i = 0; i < b.length; i++) {
                    const start = b.start(i);
                    const end = b.end(i);
                    const totalBuf = end - start;
                    const behind = ct - start;
                    const ahead = end - ct;
                    const threshold = aggressive ? 2 : (totalBuf > 180 ? 10 : 30);
                    if (behind > threshold) {
                        evicting = true;
                        const keepBehind = aggressive ? 0 : Math.min(10, totalBuf * 0.3);
                        const removeEnd = ct - keepBehind;
                        sb.remove(start, Math.max(removeEnd, start + 1));
                        _plog('[MSE] Evicted:', start.toFixed(1), '-', removeEnd.toFixed(1), '(total:', totalBuf.toFixed(1), 's)');
                        return true;
                    }
                }
            } catch(e) { _pwarn('[MSE] Evict error:', e.message); }
            return false;
        };

        const forceEvictAll = () => {
            if (evicting) return;
            if (evict(videoSB, true)) return;
            if (audioSB && evict(audioSB, true)) return;
        };

        const flushVideo = () => {
            if (videoAppending || !videoSB || videoSB.updating || videoSegQueue.length === 0) return;
            if (!videoInitAppended && videoInitSegment) {
                videoAppending = true;
                videoSB.appendBuffer(videoInitSegment);
                videoInitAppended = true;
                _plog('[MSE] Video init appended');
                return;
            }
            videoAppending = true;
            const seg = videoSegQueue.shift();
            try { videoSB.appendBuffer(seg); }
            catch(e) {
                _perr('[MSE] Video append error:', e.message);
                videoAppending = false;
                if (e.name === 'QuotaExceededError') {
                    _pwarn('[MSE] Video buffer full, evicting...');
                    pendingEviction = true;
                    if (!videoSB.updating && !evicting) forceEvictAll();
                }
            }
        };

        const flushAudio = () => {
            if (!audioSB || audioAppending || audioSB.updating || audioSegQueue.length === 0) return;
            if (!audioInitAppended && audioInitSegment) {
                audioAppending = true;
                audioSB.appendBuffer(audioInitSegment);
                audioInitAppended = true;
                _plog('[MSE] Audio init appended');
                return;
            }
            audioAppending = true;
            const seg = audioSegQueue.shift();
            try { audioSB.appendBuffer(seg); }
            catch(e) {
                _perr('[MSE] Audio append error:', e.message);
                audioAppending = false;
                if (e.name === 'QuotaExceededError') {
                    _pwarn('[MSE] Audio buffer full, evicting...');
                    pendingEviction = true;
                    if (!audioSB.updating && !evicting) forceEvictAll();
                }
            }
        };

        try {
            mp4boxFile = MP4Box.createFile();

            mp4boxFile.onReady = (info) => {
                ready = true;
                if (this._mseWatchdog) { clearInterval(this._mseWatchdog); this._mseWatchdog = null; }
                _plog('[MSE] MP4Box ready:', JSON.stringify(info).substring(0, 500));
                const vt = info.videoTracks && info.videoTracks.length > 0 ? info.videoTracks[0] : null;
                const at = info.audioTracks && info.audioTracks.length > 0 ? info.audioTracks[0] : null;

                if (!vt) {
                    _perr('[MSE] No video track found');
                    this.showErrorMessage('Nenhuma faixa de video encontrada.');
                    cleanup();
                    return;
                }

                this.populateMSEQualityInfo(info);

                let vc = vt.codec || (vt.info && vt.info.codec) || 'avc1.64001f';
                const videoMime = 'video/mp4; codecs="' + vc + '"';
                _plog('[MSE] Video MIME:', videoMime);

                if (!MediaSource.isTypeSupported(videoMime)) {
                    _pwarn('[MSE] Not supported:', videoMime);
                    const isHevc = /^hvc|^hev|^hev1/i.test(vc) || vc.indexOf('hvc1') === 0;
                    this.showErrorMessage(isHevc
                        ? 'Video HEVC (H.265) nao suportado por este navegador. Tente outro canal ou qualidade.'
                        : 'Formato nao suportado pelo navegador.');
                    cleanup();
                    return;
                }

                let audioMime = '';
                let ac = '';
                if (at) {
                    ac = at.codec || (at.info && at.info.codec) || 'mp4a.40.2';
                    audioMime = 'audio/mp4; codecs="' + ac + '"';
                    if (!MediaSource.isTypeSupported(audioMime)) {
                        _pwarn('[MSE] Audio MIME not supported:', audioMime);
                        audioMime = '';
                    }
                }

                const trackIds = [vt.id];
                if (at && audioMime) trackIds.push(at.id);

                mp4boxFile.onSegment = (trackId, user, buffer, sampleNum, isLast) => {
                    const isVid = trackId === vt.id;
                    const queue = isVid ? videoSegQueue : audioSegQueue;
                    queue.push(new Uint8Array(buffer));
                    if (isVid) flushVideo();
                    else flushAudio();
                };

                for (const tid of trackIds) {
                    mp4boxFile.setSegmentOptions(tid, null, { nbSamples: 100 });
                }

                const initSegs = mp4boxFile.initializeSegmentation();
                _plog('[MSE] Init segments:', initSegs.length);
                for (const seg of initSegs) {
                    if (seg.id === vt.id) { videoInitSegment = seg.buffer; _plog('[MSE] Video init:', seg.buffer.byteLength); }
                    else if (at && seg.id === at.id) { audioInitSegment = seg.buffer; _plog('[MSE] Audio init:', seg.buffer.byteLength); }
                }

                mp4boxFile.start();

                mediaSource = new MediaSource();
                this.videoEl.src = URL.createObjectURL(mediaSource);

                mediaSource.addEventListener('sourceopen', () => {
                    _plog('[MSE] SourceOpen');
                    try {
                        videoSB = mediaSource.addSourceBuffer(videoMime);
                        videoSB.onupdateend = () => { videoAppending = false; evicting = false; if (pendingEviction) { pendingEviction = false; forceEvictAll(); } flushVideo(); };

                        if (audioMime) {
                            audioSB = mediaSource.addSourceBuffer(audioMime);
                            audioSB.onupdateend = () => { audioAppending = false; evicting = false; if (pendingEviction) { pendingEviction = false; forceEvictAll(); } flushAudio(); };
                            _plog('[MSE] Audio SourceBuffer created');
                        }

                        this.videoEl.onloadeddata = () => {
                            this.loader.classList.add('hidden');
                            this.smartPlay();
                            this.resumeIfPossible();
                        };
                        this.videoEl.oncanplay = () => {
                            this.loader.classList.add('hidden');
                        };
                        this.videoEl.onerror = () => {
                            const err = this.videoEl.error;
                            _perr('[MSE] Video error:', err?.code, err?.message);
                            this.showErrorMessage('Erro ao reproduzir.');
                            cleanup();
                        };

                        evictionTimer = setInterval(() => {
                            if (aborted) { clearInterval(evictionTimer); return; }
                            evict(videoSB, false);
                            if (audioSB) evict(audioSB, false);
                        }, 2000);
                        this._mseEvictionTimer = evictionTimer;

                        flushVideo();
                        flushAudio();
                    } catch(e) {
                        _perr('[MSE] sourceopen error:', e.message);
                        this.showErrorMessage('Erro ao configurar player.');
                        cleanup();
                    }
                });

                mediaSource.addEventListener('error', () => {
                    _perr('[MSE] MediaSource error');
                    cleanup();
                });
            };

            mp4boxFile.onError = (e) => {
                _perr('[MSE] MP4Box error:', e);
            };

            _plog('[MSE] Starting fetch...');
            this.mseAbortController = new AbortController();
            let chunkCount = 0;

            const waitForDrain = () => {
                if (videoSegQueue.length < 15 && audioSegQueue.length < 15) return Promise.resolve();
                _plog('[MSE] Throttling fetch: vq:', videoSegQueue.length, 'aq:', audioSegQueue.length);
                return new Promise(resolve => {
                    const check = () => {
                        if (aborted || (videoSegQueue.length < 5 && audioSegQueue.length < 5)) resolve();
                        else setTimeout(check, 100);
                    };
                    check();
                });
            };

            let ready = false;
            let fedBytes = 0;
            const watchdog = setInterval(() => {
                if (aborted || ready) { clearInterval(watchdog); return; }
                if (fedBytes > 60 * 1024 * 1024) {
                    _perr('[MSE] No moov after 60MB — aborting');
                    this.showErrorMessage('Formato nao suportado (sem moov/MP4).');
                    cleanup();
                    clearInterval(watchdog);
                }
            }, 3000);
            this._mseWatchdog = watchdog;

            fetch(url, { signal: this.mseAbortController.signal }).then((response) => {
                if (!response.ok) throw new Error('HTTP ' + response.status);
                const ctype = (response.headers.get('content-type') || '').toLowerCase();
                if (ctype.includes('matroska') || ctype.includes('mkv')) {
                    _perr('[MSE] MKV content-type detected — attempting mpegts fallback');
                    cleanup();
                    this.tryMkvFallback(url);
                    return;
                }
                const reader = response.body.getReader();
                let firstChunk = true;
                const pump = () => {
                    return reader.read().then(({ done, value }) => {
                        if (done || aborted) {
                            _plog('[MSE] Fetch complete, segments queued:', videoSegQueue.length);
                            if (mp4boxFile) { try { mp4boxFile.flush(); } catch(e) {} }
                            return;
                        }
                        if (firstChunk) {
                            firstChunk = false;
                            if (value.length >= 4 && value[0] === 0x1A && value[1] === 0x45 && value[2] === 0xDF && value[3] === 0xA3) {
                                _perr('[MSE] EBML/MKV magic detected — attempting mpegts fallback');
                                cleanup();
                                this.tryMkvFallback(url);
                                return;
                            }
                        }
                        const buf = new ArrayBuffer(value.byteLength);
                        new Uint8Array(buf).set(value);
                        try {
                            buf.fileStart = fileOffset;
                            mp4boxFile.appendBuffer(buf);
                            fileOffset += value.byteLength;
                            fedBytes = fileOffset;
                            chunkCount++;
                            if (chunkCount % 100 === 0) {
                                _plog('[MSE] Fed', chunkCount, 'chunks,', fileOffset, 'bytes, vq:', videoSegQueue.length, 'aq:', audioSegQueue.length);
                            }
                        } catch(e) {
                            _perr('[MSE] mp4box error:', e?.message || e, 'offset:', fileOffset);
                        }
                        return waitForDrain().then(pump);
                    });
                };
                return pump();
            }).catch((e) => {
                if (e.name === 'AbortError' || aborted) return;
                _perr('[MSE] Fetch error:', e.message);
                const corsFail = /Failed to fetch|NetworkError|CORS/i.test(e.message || '');
                this.showErrorMessage(corsFail
                    ? 'Sem acesso ao stream (CORS/rede). Tente outro filme ou canal.'
                    : 'Erro ao carregar video.');
                cleanup();
            });

        } catch(e) {
            _perr('[MSE] Init error:', e.message);
            this.showErrorMessage('Erro ao inicializar player MSE.');
            cleanup();
        }
    }

    isMkvUrl(u) {
        return /\.mkv(\?|$)/i.test(u || this.currentUrl || '');
    }

    toHlsUrl(u) {
        if (!u) return u;
        if (/\.ts(?=\?|$)/i.test(u)) return u.replace(/\.ts(?=\?|$)/i, '.m3u8');
        return u;
    }

    showMkvError() {
        this._inMkvFallback = false;
        this.showErrorMessage('Não foi possível reproduzir este vídeo (MKV). Formato MKV não suportado pelo navegador.');
    }

    tryMkvFallback(url) {
        if (this._mkvMpegtsTried || this._vodMpegtsTried) { this.showMkvError(); return; }
        if (!(typeof mpegts !== 'undefined' && mpegts.isSupported())) { this.showMkvError(); return; }
        this._mkvMpegtsTried = true;
        this._inMkvFallback = true;
        _plog('[MKV] Native/MSE failed, attempting mpegts...');
        this.playMpegts(url);
        if (this._mkvMpegtsTimer) clearTimeout(this._mkvMpegtsTimer);
        this._mkvMpegtsTimer = setTimeout(() => {
            if (this.videoEl && this.videoEl.readyState < 2 && this.currentUrl === url) {
                this.destroyMpegts();
                this.showMkvError();
            }
        }, 10000);
        const clear = () => { if (this._mkvMpegtsTimer) { clearTimeout(this._mkvMpegtsTimer); this._mkvMpegtsTimer = null; } };
        this.videoEl.addEventListener('loadedmetadata', clear, { once: true });
        this.videoEl.addEventListener('playing', clear, { once: true });
    }

    updateBandwidthDisplay(bps) {
        const el = document.getElementById('player-bandwidth');
        if (!el || !bps) return;
        el.textContent = bps >= 1000000 ? (bps / 1000000).toFixed(1) + ' Mbps' : Math.round(bps / 1000) + ' kbps';
    }

    playDirect(url) {
        this.loader.classList.remove('hidden');
        this.videoEl.src = url;
        this.videoEl.load();
        this.videoEl.onloadeddata = () => {
            this.loader.classList.add('hidden');
            this.smartPlay();
            this.populateNativeSubtitles();
            this.populateNativeQualityInfo();
        };
        this.videoEl.oncanplay = () => {
            if (!this.loader.classList.contains('hidden')) {
                this.loader.classList.add('hidden');
                this.smartPlay();
            }
        };
        this.videoEl.onerror = () => {
            const err = this.videoEl.error;
            _pwarn('[DIRECT] Error:', err?.code, err?.message);
            if (!this.loader.classList.contains('hidden')) {
                this.showErrorMessage('Erro ao carregar video.');
            }
        };
    }

    playNativeHLS(url) {
        this.videoEl.src = url;
        this.videoEl.addEventListener('loadedmetadata', () => {
            this.loader.classList.add('hidden');
            this.smartPlay();
            this.resumeIfPossible();
            this.populateNativeSubtitles();
            this.populateNativeQualityInfo();
        }, { once: true });
        this.videoEl.addEventListener('error', () => {
            this.showErrorMessage('Erro ao carregar stream');
        }, { once: true });
    }

    destroyMpegts() {
        if (this.mpegtsPlayer) {
            try {
                this.mpegtsPlayer.pause();
                this.mpegtsPlayer.unload();
                this.mpegtsPlayer.detachMediaElement();
                this.mpegtsPlayer.destroy();
            } catch(e) {}
            this.mpegtsPlayer = null;
        }
    }

    _reportLive(ok) {
        if (!this.isLive || !this.currentStreamId || this._liveStatusSent === true && ok) return;
        if (ok) this._liveStatusSent = true;
        try {
            window.dispatchEvent(new CustomEvent('opentv:live-status', { detail: { streamId: String(this.currentStreamId), ok: !!ok } }));
        } catch (e) {}
    }

    // === Reconexao automatica de canais ao vivo ===
    // O proxy edge corta conexoes longas (~60s). Quando o stream morre ou congela,
    // reconecta no mesmo engine — o buffer maior (15s) cobre o gap.
    _onLiveEnded() {
        if (!this.isLive || !this.currentUrl || this._inMkvFallback || this._vodMpegtsTried) return;
        this._restartLive();
    }

    _restartLive() {
        if (!this.isLive || !this.currentUrl) return;
        const now = Date.now();
        // anti-loop: se a ultima reconexao foi ha menos de 4s, nao martela —
        // mas REAGENDA em vez de descartar (descartar = stream morto pra sempre)
        if (this._lastRestartAt && now - this._lastRestartAt < 4000) {
            clearTimeout(this._reconnectTimer);
            const delay = 4000 - (now - this._lastRestartAt) + 100;
            this._reconnectTimer = setTimeout(() => this._restartLive(), delay);
            return;
        }
        this._lastRestartAt = now;
        this._stallLastTime = 0;
        this._stallStrikes = 0;
        clearTimeout(this._reconnectTimer);
        this._restartCount = (this._restartCount || 0) + 1;
        if (this._restartCount > 60) { _pwarn('[LIVE] Muitas reconexoes — desistindo'); return; }
        _plog('[LIVE] Reconectando stream (' + this._restartCount + ')...');
        this._cameFromHls = false;
        this._liveStatusSent = false;
        this.destroyMpegts();
        if (this.hls) { try { this.hls.destroy(); } catch (e) {} this.hls = null; }
        const engine = this._lastLiveEngine || 'mpegts';
        const url = this.currentUrl;
        if (engine === 'hls') {
            this.playHLS(this.toHlsUrl(url));
        } else {
            this.playMpegts(url);
        }
    }

    // play com fallback de autoplay: mobile bloquea play com som fora do gesto do usuario.
    // Se bloquear, inicia MUTADO (permitido) e avisa pra tocar no alto-falante.
    smartPlay() {
        if (!this.videoEl) return;
        let p;
        try { p = this.videoEl.play(); } catch (e) { p = null; }
        if (!p || !p.catch) return;
        p.catch(err => {
            const name = err && err.name;
            if (name === 'NotAllowedError' || name === 'AbortError' || name === 'NotAllowedErrorDOMException') {
                this.videoEl.muted = true;
                if (this.btnMute) this.btnMute.innerHTML = '<i class="fas fa-volume-mute"></i>';
                let p2;
                try { p2 = this.videoEl.play(); } catch (e) { p2 = null; }
                if (p2 && p2.catch) p2.catch(() => {});
                try { window.dispatchEvent(new CustomEvent('opentv:toast', { detail: { msg: 'Toque no alto-falante (🔇) para ativar o som', type: 'info' } })); } catch (e) {}
            }
        });
    }

    // Canal HEVC sem suporte no navegador: pede pro app trocar pela versao H.264 do canal
    _hevcFallback() {
        try {
            window.dispatchEvent(new CustomEvent('opentv:hevc-fallback', { detail: { streamId: String(this.currentStreamId || ''), title: this.currentTitle || '', type: this.currentType || 'live' } }));
        } catch (e) {}
    }

    showErrorMessage(msg) {
        this.loader.classList.add('hidden');
        this.errorText.textContent = msg;
        this.errorBox.classList.remove('hidden');
        this._reportLive(false);
    }

    retry() {
        this.errorBox.classList.add('hidden');
        this.retryCount++;
        this.mediaRecoverCount = 0;
        this.playAttempt = 0;
        this._mkvMpegtsTried = false;
        this._cameFromHls = false;
        this._vodMpegtsTried = false;
        this._inMkvFallback = false;
        if (this._mkvMpegtsTimer) { clearTimeout(this._mkvMpegtsTimer); this._mkvMpegtsTimer = null; }
        this.loader.classList.remove('hidden');
        this.startStream(this.currentUrl);
    }

    togglePlay() {
        if (!this.videoEl) return;
        if (this.videoEl.paused) {
            this.smartPlay();
            if (this.btnPlayPause) this.btnPlayPause.innerHTML = '<i class="fas fa-pause"></i>';
        } else {
            this.videoEl.pause();
            if (this.btnPlayPause) this.btnPlayPause.innerHTML = '<i class="fas fa-play"></i>';
        }
        this.updateCenterPlayBtn();
    }

    toggleFullscreen() {
        const el = this.container || this.videoEl;
        const fsEl = document.fullscreenElement || document.webkitFullscreenElement || document.msFullscreenElement;
        if (!fsEl) {
            const req = el.requestFullscreen || el.webkitRequestFullscreen || el.msRequestFullscreen;
            if (req) {
                try { const p = req.call(el); if (p && p.catch) p.catch(() => this._iosFallbackFs()); }
                catch (e) { this._iosFallbackFs(); }
            } else {
                this._iosFallbackFs();
            }
        } else {
            const exit = document.exitFullscreen || document.webkitExitFullscreen || document.msExitFullscreen;
            if (exit) { try { const p = exit.call(document); if (p && p.catch) p.catch(() => {}); } catch (e) {} }
        }
        setTimeout(() => this._updateFsIcon(), 120);
    }

    _iosFallbackFs() {
        const v = this.videoEl;
        if (v && v.webkitEnterFullscreen) { try { v.webkitEnterFullscreen(); return; } catch (e) {} }
        if (v && v.webkitSupportsFullscreen) { try { v.webkitSetPresentationMode && v.webkitSetPresentationMode('fullscreen'); } catch (e) {} }
    }

    _updateFsIcon() {
        const fsEl = document.fullscreenElement || document.webkitFullscreenElement;
        if (this.btnFullscreen) {
            this.btnFullscreen.innerHTML = fsEl ? '<i class="fas fa-compress"></i>' : '<i class="fas fa-expand"></i>';
        }
    }

    toggleMute() {
        this.videoEl.muted = !this.videoEl.muted;
        if (this.btnMute) this.btnMute.innerHTML = this.videoEl.muted ? '<i class="fas fa-volume-mute"></i>' : '<i class="fas fa-volume-up"></i>';
    }

    setVolume(val) {
        this.videoEl.volume = val / 100;
        this.videoEl.muted = val == 0;
        if (this.btnMute) this.btnMute.innerHTML = this.videoEl.muted ? '<i class="fas fa-volume-mute"></i>' : '<i class="fas fa-volume-up"></i>';
    }

    skip(seconds) {
        if (this.isLive) return;
        if (!this.videoEl || !isFinite(this.videoEl.duration)) return;
        this.videoEl.currentTime = Math.max(0, Math.min(this.videoEl.duration, this.videoEl.currentTime + seconds));
        this.updateTime();
    }

    updateCenterPlayBtn() {
        if (!this.btnPlayCenter) return;
        if (this.videoEl && !this.videoEl.paused) {
            this.btnPlayCenter.innerHTML = '<i class="fas fa-pause"></i>';
        } else {
            this.btnPlayCenter.innerHTML = '<i class="fas fa-play"></i>';
        }
    }

    toggleSettings() {
        if (!this.settingsPanel) return;
        this.settingsPanel.classList.toggle('hidden');
    }

    hideSettingsPanel() {
        if (this.settingsPanel) this.settingsPanel.classList.add('hidden');
    }

    populateHLSSubtitles() {
        if (!this.subtitleOptions || !this.hls) return;
        const tracks = this.hls.subtitleTracks || [];
        if (tracks.length === 0) {
            this.subtitleOptions.innerHTML = '<div class="settings-option-text">Nenhuma legenda disponivel</div>';
            return;
        }
        let html = '<div class="settings-option" data-subtitle-track="-1"><span class="settings-option-label">Desligado</span></div>';
        tracks.forEach((t, i) => {
            const active = this.hls.subtitleTrack === i;
            html += '<div class="settings-option' + (active ? ' active' : '') + '" data-subtitle-track="' + i + '"><span class="settings-option-label">' + (t.name || t.lang || 'Legenda ' + (i + 1)) + '</span></div>';
        });
        this.subtitleOptions.innerHTML = html;
        this.subtitleOptions.querySelectorAll('.settings-option').forEach(el => {
            el.addEventListener('click', () => {
                const idx = parseInt(el.dataset.subtitleTrack);
                this.hls.subtitleTrack = idx;
                this.hls.subtitleDisplay = idx >= 0;
                this.subtitleOptions.querySelectorAll('.settings-option').forEach(o => o.classList.remove('active'));
                el.classList.add('active');
            });
        });
    }

    populateHLSAudioTracks() {
        if (!this.audioOptions || !this.hls) return;
        const tracks = this.hls.audioTracks || [];
        if (tracks.length <= 1) {
            this.audioOptions.innerHTML = '<div class="settings-option active"><span class="settings-option-label">Padrao</span></div>';
            return;
        }
        let html = '';
        tracks.forEach((t, i) => {
            const active = this.hls.audioTrack === i;
            html += '<div class="settings-option' + (active ? ' active' : '') + '" data-audio-track="' + i + '"><span class="settings-option-label">' + (t.name || t.lang || 'Audio ' + (i + 1)) + '</span></div>';
        });
        this.audioOptions.innerHTML = html;
        this.audioOptions.querySelectorAll('.settings-option').forEach(el => {
            el.addEventListener('click', () => {
                const idx = parseInt(el.dataset.audioTrack);
                this.hls.audioTrack = idx;
                this.audioOptions.querySelectorAll('.settings-option').forEach(o => o.classList.remove('active'));
                el.classList.add('active');
            });
        });
    }

    populateHLSQualityInfo() {
        if (!this.qualityOptions || !this.hls) return;
        let html = '<div class="settings-option-text">HLS Adaptive</div>';
        const levels = this.hls.levels || [];
        levels.forEach((l, i) => {
            const active = this.hls.currentLevel === i;
            const label = l.height ? l.height + 'p' : 'Nivel ' + i;
            const bitrate = l.bitrate ? ' (' + Math.round(l.bitrate / 1000) + ' kbps)' : '';
            html += '<div class="settings-option' + (active ? ' active' : '') + '" data-quality="' + i + '"><span class="settings-option-label">' + label + bitrate + '</span></div>';
        });
        html += '<div class="settings-option' + (this.hls.currentLevel === -1 ? ' active' : '') + '" data-quality="-1"><span class="settings-option-label">Automatico</span></div>';
        this.qualityOptions.innerHTML = html;
        this.qualityOptions.querySelectorAll('.settings-option').forEach(el => {
            el.addEventListener('click', () => {
                const idx = parseInt(el.dataset.quality);
                this.hls.currentLevel = idx;
                this.qualityOptions.querySelectorAll('.settings-option').forEach(o => o.classList.remove('active'));
                el.classList.add('active');
            });
        });
    }

    populateNativeSubtitles() {
        if (!this.subtitleOptions) return;
        const tracks = this.videoEl.textTracks;
        let count = 0;
        for (let i = 0; i < tracks.length; i++) {
            if (tracks[i].kind === 'subtitles' || tracks[i].kind === 'captions') count++;
        }
        if (count === 0) {
            this.subtitleOptions.innerHTML = '<div class="settings-option-text">Nenhuma legenda disponivel</div>';
            return;
        }
        let activeIdx = -1;
        let idx = 0;
        for (let i = 0; i < tracks.length; i++) {
            if (tracks[i].kind === 'subtitles' || tracks[i].kind === 'captions') {
                if (tracks[i].mode === 'showing') activeIdx = idx;
                idx++;
            }
        }
        let html = '<div class="settings-option' + (activeIdx === -1 ? ' active' : '') + '" data-subtitle-track="-1"><span class="settings-option-label">Desligado</span></div>';
        idx = 0;
        for (let i = 0; i < tracks.length; i++) {
            if (tracks[i].kind === 'subtitles' || tracks[i].kind === 'captions') {
                const trackIdx = idx;
                html += '<div class="settings-option' + (trackIdx === activeIdx ? ' active' : '') + '" data-subtitle-track="' + trackIdx + '"><span class="settings-option-label">' + (tracks[i].label || tracks[i].language || 'Legenda ' + (trackIdx + 1)) + '</span></div>';
                idx++;
            }
        }
        this.subtitleOptions.innerHTML = html;
        this.subtitleOptions.querySelectorAll('.settings-option').forEach(el => {
            el.addEventListener('click', () => {
                const targetIdx = parseInt(el.dataset.subtitleTrack);
                let currentIdx = 0;
                for (let i = 0; i < tracks.length; i++) {
                    if (tracks[i].kind === 'subtitles' || tracks[i].kind === 'captions') {
                        tracks[i].mode = currentIdx === targetIdx ? 'showing' : 'hidden';
                        currentIdx++;
                    }
                }
                this.subtitleOptions.querySelectorAll('.settings-option').forEach(o => o.classList.remove('active'));
                el.classList.add('active');
            });
        });
    }

    populateNativeQualityInfo() {
        if (!this.qualityOptions) return;
        const src = this.videoEl.src || '';
        let html = '<div class="settings-option-text">Reproduzindo diretamente</div>';
        this.qualityOptions.innerHTML = html;
    }

    populateMSEQualityInfo(info) {
        if (!this.qualityOptions) return;
        const vt = info.videoTracks && info.videoTracks[0];
        let html = '<div class="settings-option-text">MSE Playback</div>';
        if (vt) {
            const w = vt.track_width || '';
            const h = vt.track_height || '';
            if (w && h) html += '<div class="settings-option-text">' + w + 'x' + h + '</div>';
            if (vt.bitrate) html += '<div class="settings-option-text">' + Math.round(vt.bitrate / 1000) + ' kbps</div>';
        }
        const at = info.audioTracks && info.audioTracks[0];
        if (at) {
            const label = at.name || at.language || 'Audio';
            html += '<div class="settings-option active"><span class="settings-option-label">' + label + '</span></div>';
        }
        if (!this.hls) {
            this.audioOptions.innerHTML = '<div class="settings-option active"><span class="settings-option-label">Padrao</span></div>';
        }
this.qualityOptions.innerHTML = html;
    }

    // Preenche o painel de qualidade com variantes de canal (SD/HD/FHD/4K + provedor)
    populateStreamVariants() {
        if (!this.qualityOptions || !this.currentVariants || !this.currentVariants.length) return;
        const currentId = this.currentStreamId;
        const defaultId = (window.Providers && window.Providers.DEFAULT_ID) || 'telefunplay';
        let html = '<div class="settings-option-title"><i class="fas fa-tv"></i> Qualidade / Fonte</div>';
        this.currentVariants.forEach((v, idx) => {
            const isActive = String(v.stream_id) === String(currentId);
            const qLabel = this.qualityLabel ? this.qualityLabel(v.name) : v.name;
            const prov = v.provider && v.provider !== defaultId ? v.provider : 'Padrão';
            html += '<div class="settings-option' + (isActive ? ' active' : '') + '" data-variant-idx="' + idx + '">' +
                '<span class="settings-option-label">' + qLabel + '</span>' +
                '<span class="variant-provider">' + prov + '</span>' +
                '</div>';
        });
        this.qualityOptions.innerHTML = html;
        this.qualityOptions.querySelectorAll('.settings-option[data-variant-idx]').forEach(el => {
            el.addEventListener('click', () => {
                const idx = parseInt(el.dataset.variantIdx, 10);
                const variant = this.currentVariants[idx];
                if (!variant) return;
                // dispa evento para app.js trocar o stream
                const ev = new CustomEvent('opentv:switch-variant', { detail: { streamId: variant.stream_id, variants: this.currentVariants } });
                document.dispatchEvent(ev);
            });
        });
    }

    // Helper para label de qualidade (reutiliza lógica do app.js se disponível)
    qualityLabel(name) {
        const n = String(name || '');
        const hevc = /h\.?\s*265|hevc|x265|\.h265/i.test(n);
        let q = 'SD';
        if (/\b(2160p|4k|uhd|ultra hd)\b/i.test(n)) q = '4K';
        else if (/\b(1080p|fhd|full hd)\b/i.test(n)) q = 'FHD';
        else if (/\b(720p|hd)\b/i.test(n)) q = 'HD';
        else if (/\b(sd)\b/i.test(n)) q = 'SD';
        return hevc ? q + ' (H265)' : q;
    }

    onSeekStart(e) {
        if (this.isLive) return;
        if (!this.videoEl || !isFinite(this.videoEl.duration)) return;
        this.seekDragging = true;
        this.seekToPosition(e);
    }

    onSeekDragMove(e) {
        if (!this.seekDragging) return;
        if (this.isLive) return;
        this.seekToPosition(e);
    }

    onSeekDragEnd(e) {
        if (!this.seekDragging) return;
        this.seekDragging = false;
    }

    onSeekHover(e) {
        if (this.isLive) return;
        if (!this.seekBarWrapper || !this.seekTooltip) return;
        if (!isFinite(this.videoEl?.duration)) return;
        const rect = this.seekBarWrapper.getBoundingClientRect();
        const pct = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
        const time = pct * this.videoEl.duration;
        this.seekTooltip.textContent = this.fmt(time);
        this.seekTooltip.style.left = (pct * 100) + '%';
        this.seekTooltip.style.opacity = '1';
    }

    onSeekHoverEnd() {
        if (this.seekTooltip) this.seekTooltip.style.opacity = '0';
    }

    seekToPosition(e) {
        if (!this.seekBarWrapper || !this.videoEl) return;
        const rect = this.seekBarWrapper.getBoundingClientRect();
        const pct = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
        if (isFinite(this.videoEl.duration)) {
            this.videoEl.currentTime = pct * this.videoEl.duration;
            this.updateTime();
        }
    }

    updateSeekUI() {
        if (this.isLive || !this.videoEl) {
            if (this.seekProgress) this.seekProgress.style.width = '0%';
            if (this.seekBuffered) this.seekBuffered.style.width = '0%';
            if (this.seekThumb) this.seekThumb.style.left = '0%';
            if (this.timeCurrent) this.timeCurrent.textContent = '';
            if (this.timeDuration) this.timeDuration.textContent = '';
            return;
        }
    }

    updateTime() {
        if (this.isLive) {
            if (this.timeDisplay) this.timeDisplay.textContent = '';
            this.updateSeekUI();
            return;
        }
        const c = this.videoEl.currentTime || 0;
        const d = this.videoEl.duration || 0;
        if (this.timeDisplay) this.timeDisplay.textContent = this.fmt(c) + ' / ' + this.fmt(d);
        if (this.timeCurrent) this.timeCurrent.textContent = this.fmt(c);
        if (this.timeDuration) this.timeDuration.textContent = this.fmt(d);

        if (this.seekProgress && isFinite(d) && d > 0) {
            this.seekProgress.style.width = ((c / d) * 100) + '%';
        }
        if (this.seekThumb && isFinite(d) && d > 0) {
            this.seekThumb.style.left = ((c / d) * 100) + '%';
        }

        if (this.seekBuffered && this.videoEl.buffered.length > 0 && isFinite(d) && d > 0) {
            const buffEnd = this.videoEl.buffered.end(this.videoEl.buffered.length - 1);
            this.seekBuffered.style.width = ((buffEnd / d) * 100) + '%';
        }

        this.updateCenterPlayBtn();
    }

    fmt(s) {
        if (isNaN(s)) return '0:00';
        const h = Math.floor(s / 3600), m = Math.floor((s % 3600) / 60), sec = Math.floor(s % 60);
        return h > 0 ? `${h}:${String(m).padStart(2, '0')}:${String(sec).padStart(2, '0')}` : `${m}:${String(sec).padStart(2, '0')}`;
    }

    stop() {
        this.stopProgressSaver();
        this.cleanup();
        this.currentStreamId = null;
        this.resumeAt = 0;
        this.container.classList.add('hidden');
        document.body.style.overflow = '';
    }

    cleanup() {
        if (!this.isLive && this.currentStreamId != null && this.videoEl && isFinite(this.videoEl.duration) && this.videoEl.duration > 0 && typeof WatchStore !== 'undefined') {
            WatchStore.saveProgress(this.currentType, this.currentStreamId, this.videoEl.currentTime, this.videoEl.duration, this.currentTitle);
        }
        this.stopProgressSaver();
        this._pendingResume = 0;
        this.resumeAt = 0;
        if (this._mseEvictionTimer) { clearInterval(this._mseEvictionTimer); this._mseEvictionTimer = null; }
        if (this.mseFallbackTimer) { clearTimeout(this.mseFallbackTimer); this.mseFallbackTimer = null; }
        if (this._mkvMpegtsTimer) { clearTimeout(this._mkvMpegtsTimer); this._mkvMpegtsTimer = null; }
        this._inMkvFallback = false;
        clearTimeout(this._reconnectTimer);
        this.currentUrl = '';
        if (this.mseAbortController) { this.mseAbortController.abort(); this.mseAbortController = null; }
        this.loader.classList.add('hidden');
        this.errorBox.classList.add('hidden');
        if (this.videoEl) {
            try { if (window.SubtitleStore) SubtitleStore.clearTracks(this.videoEl); } catch (e) {}
            this.videoEl.pause();
            this.videoEl.removeAttribute('src');
            this.videoEl.load();
        }
        if (this.hls) { try { this.hls.destroy(); } catch (e) {} this.hls = null; }
        this.destroyMpegts();
        if (this.btnPlayPause) this.btnPlayPause.innerHTML = '<i class="fas fa-play"></i>';
        this.updateCenterPlayBtn();
        this.hideSettingsPanel();
        this.hideResumeModal();
        this.updateSeekUI();
    }
}

const player = new VideoPlayer();
