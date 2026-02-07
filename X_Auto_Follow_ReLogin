// ==UserScript==
// @name         X.com Auto Follow + Auto Re-Login
// @namespace    http://tampermonkey.net/
// @version      2.1
// @description  Automatically follow users when viewing their posts and auto re-login when forced logout occurs
// @author       You
// @match        https://x.com/*
// @match        https://twitter.com/*
// @grant        GM_getValue
// @grant        GM_setValue
// @grant        GM_deleteValue
// @run-at       document-end
// ==/UserScript==

(function () {
    'use strict';

    // ============================================================================
    // CONFIG MODULE - Centralized Configuration
    // ============================================================================
    const CONFIG = {
        // Auto-Follow Settings
        FOLLOW: {
            RECHECK_DELAY: 2000,
            MAX_RETRIES: 2,
            BUTTON_SEARCH_ATTEMPTS: 10,
            BUTTON_SEARCH_INTERVAL: 300
        },

        // Auto Re-Login Settings
        LOGIN: {
            EMAIL_DOMAIN: '@test.com',
            USERNAME_STORAGE_KEY: 'x_auto_login_username', // GM storage key for username
            PASSWORD_STORAGE_KEY: 'x_auto_login_password', // GM storage key for password
            MAX_RETRIES: 3,
            STEP_DELAY: 1500,
            ELEMENT_WAIT_TIMEOUT: 10000,
            LOGIN_URL: 'https://x.com/i/flow/login'
        },

        // Detection Settings
        DETECTION: {
            LOGOUT_CHECK_INTERVAL: 5000,
            OBSERVER_DEBOUNCE: 500,
            PAGE_STATE_CHECK_DELAY: 1000
        },

        // Selectors (with fallbacks)
        SELECTORS: {
            emailInput: [
                'input[name="text"]',
                'input[autocomplete="username"]',
                'input[type="text"]'
            ],
            passwordInput: [
                'input[name="password"]',
                'input[type="password"]'
            ],
            nextButton: [
                '[data-testid="ocfEnterTextNextButton"]',
                'button[role="button"]'
            ],
            loginButton: [
                '[data-testid="LoginForm_Login_Button"]',
                'button[data-testid="LoginForm_Login_Button"]'
            ],
            followButton: [
                'button[data-testid*="follow"]',
                'button[role="button"]'
            ]
        },

        // Text Patterns
        TEXT: {
            follow: ['Follow', 'フォロー'],
            following: ['Following', 'フォロー中'],
            next: ['Next', '次へ'],
            login: ['Log in', 'ログイン'],
            loginPrompt: ['Log in', 'ログイン', 'Sign in', 'サインイン']
        },

        // Feature Flags
        FLAGS: {
            DRY_RUN: false,
            SAFE_STOP_ERROR_THRESHOLD: 5,
            ENABLE_AUTO_FOLLOW: true,
            ENABLE_AUTO_LOGIN: true,
            VERBOSE_LOGGING: true
        }
    };

    // ============================================================================
    // LOGGER MODULE - Unified Logging
    // ============================================================================
    const Logger = {
        _prefix: '[X Auto-Follow+Login]',

        _log(level, feature, message, ...args) {
            const timestamp = new Date().toLocaleTimeString('ja-JP');
            const prefix = `${this._prefix} [${timestamp}] [${level}] [${feature}]`;

            switch (level) {
                case 'ERROR':
                    console.error(prefix, message, ...args);
                    break;
                case 'WARN':
                    console.warn(prefix, message, ...args);
                    break;
                case 'DEBUG':
                    if (CONFIG.FLAGS.VERBOSE_LOGGING) {
                        console.log(prefix, message, ...args);
                    }
                    break;
                default:
                    console.log(prefix, message, ...args);
            }
        },

        info(feature, message, ...args) {
            this._log('INFO', feature, message, ...args);
        },

        warn(feature, message, ...args) {
            this._log('WARN', feature, message, ...args);
        },

        error(feature, message, ...args) {
            this._log('ERROR', feature, message, ...args);
        },

        debug(feature, message, ...args) {
            this._log('DEBUG', feature, message, ...args);
        }
    };

    // ============================================================================
    // DOM UTILITIES MODULE - DOM Manipulation Helpers
    // ============================================================================
    const DOMUtils = {
        /**
         * Wait for element to appear in DOM
         */
        async waitForElement(selectors, timeout = CONFIG.LOGIN.ELEMENT_WAIT_TIMEOUT) {
            const selectorArray = Array.isArray(selectors) ? selectors : [selectors];
            const startTime = Date.now();

            while (Date.now() - startTime < timeout) {
                for (const selector of selectorArray) {
                    const element = document.querySelector(selector);
                    if (element) {
                        Logger.debug('DOMUtils', `Element found: ${selector}`);
                        return element;
                    }
                }
                await this.wait(100);
            }

            Logger.warn('DOMUtils', `Element not found after ${timeout}ms:`, selectorArray);
            return null;
        },

        /**
         * Find button by text content (supports multiple languages)
         */
        findButtonByText(textOptions, exactMatch = true) {
            const buttons = document.querySelectorAll('button[role="button"], button');

            for (const button of buttons) {
                const buttonText = button.textContent.trim();

                for (const text of textOptions) {
                    if (exactMatch) {
                        if (buttonText === text) {
                            Logger.debug('DOMUtils', `Button found with text: "${text}"`);
                            return button;
                        }
                    } else {
                        if (buttonText.includes(text)) {
                            Logger.debug('DOMUtils', `Button found containing text: "${text}"`);
                            return button;
                        }
                    }
                }
            }

            Logger.debug('DOMUtils', `Button not found with texts:`, textOptions);
            return null;
        },

        /**
         * Fill input field with value (React-compatible)
         */
        async fillInput(element, value) {
            if (!element) {
                Logger.error('DOMUtils', 'Cannot fill input: element is null');
                return false;
            }

            try {
                // Focus the input
                element.focus();

                // Method 1: Use React's native setter if available
                const nativeInputValueSetter = Object.getOwnPropertyDescriptor(
                    window.HTMLInputElement.prototype,
                    'value'
                ).set;

                if (nativeInputValueSetter) {
                    nativeInputValueSetter.call(element, value);
                } else {
                    element.value = value;
                }

                // Method 2: Trigger multiple events for React
                // Input event (for controlled components)
                element.dispatchEvent(new Event('input', { bubbles: true }));

                // Change event (for form validation)
                element.dispatchEvent(new Event('change', { bubbles: true }));

                // Blur and focus to ensure validation
                element.dispatchEvent(new Event('blur', { bubbles: true }));
                element.dispatchEvent(new Event('focus', { bubbles: true }));

                // KeyDown/KeyUp events (some forms check for these)
                element.dispatchEvent(new KeyboardEvent('keydown', { bubbles: true }));
                element.dispatchEvent(new KeyboardEvent('keyup', { bubbles: true }));

                // Wait a bit for React to process
                await this.wait(300);

                Logger.debug('DOMUtils', `Filled input with value: ${value}`);
                return true;
            } catch (error) {
                Logger.error('DOMUtils', 'Error filling input:', error);
                return false;
            }
        },

        /**
         * Click element safely
         */
        async clickElement(element, description = 'element') {
            if (!element) {
                Logger.error('DOMUtils', `Cannot click ${description}: element is null`);
                return false;
            }

            if (CONFIG.FLAGS.DRY_RUN) {
                Logger.info('DOMUtils', `[DRY RUN] Would click: ${description}`);
                return true;
            }

            try {
                element.click();
                Logger.debug('DOMUtils', `Clicked: ${description}`);
                return true;
            } catch (error) {
                Logger.error('DOMUtils', `Error clicking ${description}:`, error);
                return false;
            }
        },

        /**
         * Wait utility
         */
        wait(ms) {
            return new Promise(resolve => setTimeout(resolve, ms));
        },

        /**
         * Retry utility with exponential backoff
         */
        async withRetry(fn, maxRetries, delayMs, description) {
            for (let attempt = 1; attempt <= maxRetries; attempt++) {
                try {
                    Logger.debug('DOMUtils', `Attempt ${attempt}/${maxRetries}: ${description}`);
                    const result = await fn();
                    if (result) {
                        Logger.debug('DOMUtils', `Success on attempt ${attempt}: ${description}`);
                        return result;
                    }
                } catch (error) {
                    Logger.error('DOMUtils', `Error on attempt ${attempt}: ${description}`, error);
                }

                if (attempt < maxRetries) {
                    const waitTime = delayMs * Math.pow(1.5, attempt - 1);
                    Logger.debug('DOMUtils', `Waiting ${waitTime}ms before retry...`);
                    await this.wait(waitTime);
                }
            }

            Logger.warn('DOMUtils', `Failed after ${maxRetries} attempts: ${description}`);
            return null;
        }
    };

    // ============================================================================
    // STATE DETECTION MODULE - Page State Detection
    // ============================================================================
    const StateDetection = {
        /**
         * Check if user is logged in
         */
        isLoggedIn() {
            // Check for login prompts
            const loginButtons = document.querySelectorAll('a[href="/login"], a[data-testid="loginButton"]');
            if (loginButtons.length > 0) {
                Logger.debug('StateDetection', 'Login buttons found - user is logged out');
                return false;
            }

            // Check for login text in buttons
            const hasLoginPrompt = DOMUtils.findButtonByText(CONFIG.TEXT.loginPrompt, false);
            if (hasLoginPrompt) {
                Logger.debug('StateDetection', 'Login prompt found - user is logged out');
                return false;
            }

            // Check if on login page
            if (this.isLoginPage()) {
                Logger.debug('StateDetection', 'On login page - user is logged out');
                return false;
            }

            Logger.debug('StateDetection', 'User appears to be logged in');
            return true;
        },

        /**
         * Check if current page is login flow
         */
        isLoginPage() {
            return window.location.pathname.includes('/i/flow/login') ||
                window.location.pathname.includes('/login');
        },

        /**
         * Check if current page is post detail
         */
        isPostDetailPage() {
            const url = window.location.pathname;
            const postDetailPattern = /^\/[^\/]+\/status\/\d+/;
            return postDetailPattern.test(url);
        },

        /**
         * Get current page state
         */
        getCurrentPageState() {
            if (this.isLoginPage()) {
                return 'LOGIN_PAGE';
            } else if (!this.isLoggedIn()) {
                return 'LOGGED_OUT';
            } else if (this.isPostDetailPage()) {
                return 'POST_DETAIL';
            } else {
                return 'NORMAL';
            }
        }
    };

    // ============================================================================
    // AUTO-FOLLOW FEATURE MODULE
    // ============================================================================
    const AutoFollowFeature = {
        lastProcessedUrl: '',
        isProcessing: false,

        /**
         * Find follow button
         */
        async findFollowButton() {
            return await DOMUtils.withRetry(
                () => {
                    const buttons = document.querySelectorAll('button[data-testid*="follow"], button[role="button"]');

                    for (const button of buttons) {
                        const buttonText = button.textContent.trim();

                        // Check for exact match with "Follow" or "フォロー"
                        if (CONFIG.TEXT.follow.includes(buttonText)) {
                            // Make sure it's not "Following" or "フォロー中"
                            if (!CONFIG.TEXT.following.some(text => buttonText.includes(text))) {
                                return button;
                            }
                        }
                    }
                    return null;
                },
                CONFIG.FOLLOW.BUTTON_SEARCH_ATTEMPTS,
                CONFIG.FOLLOW.BUTTON_SEARCH_INTERVAL,
                'Find follow button'
            );
        },

        /**
         * Check if already following
         */
        isAlreadyFollowing() {
            const buttons = document.querySelectorAll('button[data-testid*="follow"], button[role="button"]');

            for (const button of buttons) {
                const buttonText = button.textContent.trim();

                // Check for "Following" or "フォロー中"
                if (CONFIG.TEXT.following.some(text => buttonText === text || buttonText.includes(text))) {
                    return true;
                }
            }

            return false;
        },

        /**
         * Execute follow action
         */
        async executeFollow() {
            if (!CONFIG.FLAGS.ENABLE_AUTO_FOLLOW) {
                Logger.info('AutoFollow', 'Auto-follow is disabled');
                return;
            }

            if (this.isProcessing) {
                Logger.debug('AutoFollow', 'Already processing, skipping...');
                return;
            }

            const currentUrl = window.location.href;
            if (currentUrl === this.lastProcessedUrl) {
                Logger.debug('AutoFollow', 'Already processed this URL, skipping...');
                return;
            }

            if (!StateDetection.isPostDetailPage()) {
                Logger.debug('AutoFollow', 'Not a post detail page, skipping...');
                return;
            }

            this.isProcessing = true;
            this.lastProcessedUrl = currentUrl;

            Logger.info('AutoFollow', 'Post detail page detected:', currentUrl);

            try {
                // Check if already following
                if (this.isAlreadyFollowing()) {
                    Logger.info('AutoFollow', 'Already following this account, no action needed');
                    return;
                }

                // Search for follow button
                const followButton = await this.findFollowButton();

                if (!followButton) {
                    Logger.warn('AutoFollow', 'Follow button not found');
                    return;
                }

                Logger.info('AutoFollow', 'Found unfollowed user, attempting to follow...');

                // Click the follow button
                if (!await DOMUtils.clickElement(followButton, 'Follow button')) {
                    Logger.error('AutoFollow', 'Failed to click follow button');
                    return;
                }

                Logger.info('AutoFollow', 'Follow button clicked, waiting for confirmation...');

                // Wait before rechecking
                await DOMUtils.wait(CONFIG.FOLLOW.RECHECK_DELAY);

                // Recheck follow status
                Logger.info('AutoFollow', 'Rechecking follow status...');

                if (this.isAlreadyFollowing()) {
                    Logger.info('AutoFollow', '✓ Successfully followed user');
                    return;
                }

                // If still not following, try again
                const retryButton = await this.findFollowButton();
                if (retryButton) {
                    Logger.info('AutoFollow', 'Still not following, attempting retry...');
                    if (await DOMUtils.clickElement(retryButton, 'Follow button (retry)')) {
                        Logger.info('AutoFollow', 'Retry follow button clicked');

                        // Wait and check one more time
                        await DOMUtils.wait(CONFIG.FOLLOW.RECHECK_DELAY);

                        if (this.isAlreadyFollowing()) {
                            Logger.info('AutoFollow', '✓ Successfully followed user on retry');
                        } else {
                            Logger.warn('AutoFollow', '⚠ Follow may have failed after retry');
                        }
                    }
                } else {
                    Logger.info('AutoFollow', '✓ Follow button no longer visible (likely followed successfully)');
                }

            } catch (error) {
                Logger.error('AutoFollow', 'Error during auto-follow:', error);
            } finally {
                this.isProcessing = false;
            }
        },

        /**
         * Reset state (for URL changes)
         */
        reset() {
            this.lastProcessedUrl = '';
        }
    };

    // ============================================================================
    // AUTO RE-LOGIN FEATURE MODULE
    // ============================================================================
    const AutoLoginFeature = {
        isLoggingIn: false,
        loginAttempts: 0,

        /**
         * Get username from GM storage or window.name (fallback)
         */
        getUsername() {
            // First try GM storage
            let username = GM_getValue(CONFIG.LOGIN.USERNAME_STORAGE_KEY, '');

            // Fallback to window.name for backward compatibility
            if (!username) {
                username = window.name;
                // If found in window.name, save to GM storage for future use
                if (username) {
                    this.setUsername(username);
                    Logger.info('AutoLogin', `Migrated username from window.name to GM storage: ${username}`);
                }
            }

            if (!username) {
                Logger.warn('AutoLogin', '⚠️ ユーザ名が設定されていません');
                Logger.warn('AutoLogin', 'ヒント: コンソールで GM_setValue("x_auto_login_username", "your_username") を実行してください');
                return null;
            }

            Logger.info('AutoLogin', `Retrieved username: ${username}`);
            return username;
        },

        /**
         * Set username to GM storage (helper function)
         */
        setUsername(username) {
            GM_setValue(CONFIG.LOGIN.USERNAME_STORAGE_KEY, username);
            Logger.info('AutoLogin', `Username saved to storage: ${username}`);
        },

        /**
         * Generate email from username
         */
        generateEmail(username) {
            if (!username) return null;
            const email = `${username}${CONFIG.LOGIN.EMAIL_DOMAIN}`;
            Logger.info('AutoLogin', `Generated email: ${email}`);
            return email;
        },

        /**
         * Get password from GM storage
         */
        getPassword() {
            const password = GM_getValue(CONFIG.LOGIN.PASSWORD_STORAGE_KEY, '');
            if (!password) {
                Logger.warn('AutoLogin', '⚠️ パスワードが設定されていません');
                Logger.warn('AutoLogin', 'ヒント: コンソールで GM_setValue("x_auto_login_password", "your_password") を実行してください');
                return null;
            }
            Logger.info('AutoLogin', 'Password retrieved from storage');
            return password;
        },

        /**
         * Set password to GM storage (helper function)
         */
        setPassword(password) {
            GM_setValue(CONFIG.LOGIN.PASSWORD_STORAGE_KEY, password);
            Logger.info('AutoLogin', 'Password saved to storage');
        },

        /**
         * Navigate to login page
         */
        async navigateToLogin() {
            Logger.info('AutoLogin', 'Navigating to login page...');

            if (StateDetection.isLoginPage()) {
                Logger.info('AutoLogin', 'Already on login page');
                return true;
            }

            try {
                window.location.href = CONFIG.LOGIN.LOGIN_URL;
                await DOMUtils.wait(CONFIG.LOGIN.STEP_DELAY);
                return true;
            } catch (error) {
                Logger.error('AutoLogin', 'Error navigating to login:', error);
                return false;
            }
        },

        /**
         * Fill email and click Next
         */
        async fillEmailAndNext() {
            Logger.info('AutoLogin', 'Step 1: Filling email...');

            // Get username
            const username = this.getUsername();
            if (!username) {
                Logger.error('AutoLogin', 'Cannot proceed without username');
                return false;
            }

            // Generate email
            const email = this.generateEmail(username);
            if (!email) {
                Logger.error('AutoLogin', 'Cannot proceed without email');
                return false;
            }

            // Wait for email input
            const emailInput = await DOMUtils.waitForElement(CONFIG.SELECTORS.emailInput);
            if (!emailInput) {
                Logger.error('AutoLogin', 'Email input not found');
                return false;
            }

            // Fill email
            if (!await DOMUtils.fillInput(emailInput, email)) {
                Logger.error('AutoLogin', 'Failed to fill email');
                return false;
            }

            // Verify the value was set correctly
            await DOMUtils.wait(500);
            if (emailInput.value !== email) {
                Logger.warn('AutoLogin', 'Email value mismatch, retrying...');
                // Retry once
                if (!await DOMUtils.fillInput(emailInput, email)) {
                    Logger.error('AutoLogin', 'Failed to fill email on retry');
                    return false;
                }
            }

            Logger.info('AutoLogin', `Email filled and verified: ${email}`);
            Logger.info('AutoLogin', 'Waiting before clicking Next...');
            await DOMUtils.wait(CONFIG.LOGIN.STEP_DELAY);

            // Click Next button
            const nextButton = DOMUtils.findButtonByText(CONFIG.TEXT.next);
            if (!nextButton) {
                Logger.error('AutoLogin', 'Next button not found');
                return false;
            }

            if (!await DOMUtils.clickElement(nextButton, 'Next button')) {
                Logger.error('AutoLogin', 'Failed to click Next button');
                return false;
            }

            Logger.info('AutoLogin', '✓ Email step completed');
            await DOMUtils.wait(CONFIG.LOGIN.STEP_DELAY);
            return true;
        },

        /**
         * Handle additional authentication (username request)
         */
        async handleAdditionalAuth() {
            Logger.info('AutoLogin', 'Checking for additional authentication...');

            // Wait a bit to see if additional auth appears
            await DOMUtils.wait(CONFIG.LOGIN.STEP_DELAY);

            // Look for username input (additional auth screen)
            const additionalInput = document.querySelector('input[name="text"]');
            if (!additionalInput) {
                Logger.info('AutoLogin', 'No additional authentication required');
                return true;
            }

            Logger.info('AutoLogin', 'Additional authentication detected, filling username...');

            // Get username
            const username = this.getUsername();
            if (!username) {
                Logger.error('AutoLogin', 'Cannot proceed without username for additional auth');
                return false;
            }

            // Fill username
            if (!await DOMUtils.fillInput(additionalInput, username)) {
                Logger.error('AutoLogin', 'Failed to fill username for additional auth');
                return false;
            }

            await DOMUtils.wait(CONFIG.LOGIN.STEP_DELAY);

            // Click Next
            const nextButton = DOMUtils.findButtonByText(CONFIG.TEXT.next);
            if (!nextButton) {
                Logger.error('AutoLogin', 'Next button not found for additional auth');
                return false;
            }

            if (!await DOMUtils.clickElement(nextButton, 'Next button (additional auth)')) {
                Logger.error('AutoLogin', 'Failed to click Next button for additional auth');
                return false;
            }

            Logger.info('AutoLogin', '✓ Additional authentication completed');
            await DOMUtils.wait(CONFIG.LOGIN.STEP_DELAY);
            return true;
        },

        /**
         * Fill password and login
         */
        async fillPasswordAndLogin() {
            Logger.info('AutoLogin', 'Step 2: Filling password...');

            // Get password from storage
            const password = this.getPassword();
            if (!password) {
                Logger.error('AutoLogin', 'Cannot proceed without password');
                return false;
            }

            // Wait for password input
            const passwordInput = await DOMUtils.waitForElement(CONFIG.SELECTORS.passwordInput);
            if (!passwordInput) {
                Logger.error('AutoLogin', 'Password input not found');
                return false;
            }

            // Fill password
            if (!await DOMUtils.fillInput(passwordInput, password)) {
                Logger.error('AutoLogin', 'Failed to fill password');
                return false;
            }

            Logger.info('AutoLogin', 'Password filled, waiting before clicking Login...');
            await DOMUtils.wait(CONFIG.LOGIN.STEP_DELAY);

            // Click Login button
            const loginButton = DOMUtils.findButtonByText(CONFIG.TEXT.login);
            if (!loginButton) {
                Logger.error('AutoLogin', 'Login button not found');
                return false;
            }

            if (!await DOMUtils.clickElement(loginButton, 'Login button')) {
                Logger.error('AutoLogin', 'Failed to click Login button');
                return false;
            }

            Logger.info('AutoLogin', '✓ Password step completed, waiting for login...');
            await DOMUtils.wait(CONFIG.LOGIN.STEP_DELAY * 2);
            return true;
        },

        /**
         * Verify login success
         */
        async verifyLoginSuccess() {
            Logger.info('AutoLogin', 'Verifying login success...');

            // Wait for page to settle
            await DOMUtils.wait(CONFIG.LOGIN.STEP_DELAY);

            // Check if logged in
            if (StateDetection.isLoggedIn()) {
                Logger.info('AutoLogin', '✓ Login successful!');
                return true;
            }

            Logger.warn('AutoLogin', 'Login verification failed');
            return false;
        },

        /**
         * Execute full login flow
         */
        async executeLogin() {
            if (!CONFIG.FLAGS.ENABLE_AUTO_LOGIN) {
                Logger.info('AutoLogin', 'Auto-login is disabled');
                return false;
            }

            if (this.isLoggingIn) {
                Logger.debug('AutoLogin', 'Already logging in, skipping...');
                return false;
            }

            if (this.loginAttempts >= CONFIG.LOGIN.MAX_RETRIES) {
                Logger.error('AutoLogin', `Max login attempts (${CONFIG.LOGIN.MAX_RETRIES}) reached`);
                return false;
            }

            this.isLoggingIn = true;
            this.loginAttempts++;

            Logger.info('AutoLogin', `Starting login flow (attempt ${this.loginAttempts}/${CONFIG.LOGIN.MAX_RETRIES})...`);

            try {
                // Navigate to login if not already there
                if (!StateDetection.isLoginPage()) {
                    if (!await this.navigateToLogin()) {
                        return false;
                    }
                }

                // Step 1: Fill email and click Next
                if (!await this.fillEmailAndNext()) {
                    Logger.error('AutoLogin', 'Email step failed');
                    return false;
                }

                // Handle additional authentication if needed
                if (!await this.handleAdditionalAuth()) {
                    Logger.error('AutoLogin', 'Additional authentication failed');
                    return false;
                }

                // Step 2: Fill password and login
                if (!await this.fillPasswordAndLogin()) {
                    Logger.error('AutoLogin', 'Password step failed');
                    return false;
                }

                // Verify login success
                if (!await this.verifyLoginSuccess()) {
                    Logger.error('AutoLogin', 'Login verification failed');
                    return false;
                }

                Logger.info('AutoLogin', '✓✓✓ Full login flow completed successfully! ✓✓✓');
                this.loginAttempts = 0; // Reset on success
                return true;

            } catch (error) {
                Logger.error('AutoLogin', 'Error during login flow:', error);
                return false;
            } finally {
                this.isLoggingIn = false;
            }
        },

        /**
         * Detect logout and trigger re-login
         */
        async detectAndHandleLogout() {
            if (!StateDetection.isLoggedIn()) {
                Logger.warn('AutoLogin', 'Logout detected! Initiating auto re-login...');
                return await this.executeLogin();
            }
            return false;
        }
    };

    // ============================================================================
    // ORCHESTRATOR MODULE - Main Control Flow
    // ============================================================================
    const Orchestrator = {
        consecutiveErrors: 0,
        lastUrl: '',
        debounceTimer: null,

        /**
         * Handle URL change
         */
        async handleUrlChange() {
            clearTimeout(this.debounceTimer);
            this.debounceTimer = setTimeout(async () => {
                await this.processCurrentPage();
            }, CONFIG.DETECTION.OBSERVER_DEBOUNCE);
        },

        /**
         * Process current page based on state
         */
        async processCurrentPage() {
            try {
                const pageState = StateDetection.getCurrentPageState();
                Logger.debug('Orchestrator', `Current page state: ${pageState}`);

                // Check for /i/chat redirect
                if (window.location.pathname === '/i/chat') {
                    Logger.info('Orchestrator', 'Redirecting /i/chat to /messages...');
                    window.location.href = 'https://x.com/messages';
                    return;
                }

                switch (pageState) {
                    case 'LOGGED_OUT':
                        Logger.info('Orchestrator', 'User is logged out, attempting re-login...');
                        await AutoLoginFeature.executeLogin();
                        break;

                    case 'LOGIN_PAGE':
                        Logger.info('Orchestrator', 'On login page, executing login flow...');
                        await AutoLoginFeature.executeLogin();
                        break;

                    case 'POST_DETAIL':
                        Logger.info('Orchestrator', 'On post detail page, executing auto-follow...');
                        await AutoFollowFeature.executeFollow();
                        break;

                    case 'NORMAL':
                        Logger.debug('Orchestrator', 'Normal page, no action needed');
                        break;
                }

                this.consecutiveErrors = 0; // Reset on success

            } catch (error) {
                this.consecutiveErrors++;
                Logger.error('Orchestrator', `Error processing page (${this.consecutiveErrors} consecutive):`, error);

                if (this.consecutiveErrors >= CONFIG.FLAGS.SAFE_STOP_ERROR_THRESHOLD) {
                    Logger.error('Orchestrator', `Safe stop triggered after ${this.consecutiveErrors} consecutive errors`);
                    this.stop();
                }
            }
        },

        /**
         * Monitor URL changes
         */
        monitorUrlChanges() {
            this.lastUrl = location.href;

            const observer = new MutationObserver(() => {
                const currentUrl = location.href;
                if (currentUrl !== this.lastUrl) {
                    this.lastUrl = currentUrl;
                    Logger.info('Orchestrator', 'URL changed:', currentUrl);
                    AutoFollowFeature.reset();
                    this.handleUrlChange();
                }
            });

            observer.observe(document.body, {
                childList: true,
                subtree: true
            });

            Logger.info('Orchestrator', 'URL monitoring started');
        },

        /**
         * Periodic logout check
         */
        startLogoutMonitoring() {
            setInterval(async () => {
                if (!StateDetection.isLoginPage() && !AutoLoginFeature.isLoggingIn) {
                    await AutoLoginFeature.detectAndHandleLogout();
                }
            }, CONFIG.DETECTION.LOGOUT_CHECK_INTERVAL);

            Logger.info('Orchestrator', 'Logout monitoring started');
        },

        /**
         * Initialize the orchestrator
         */
        async init() {
            Logger.info('Orchestrator', '='.repeat(60));
            Logger.info('Orchestrator', 'X.com Auto-Follow + Auto Re-Login Script Initialized');
            Logger.info('Orchestrator', `Version: 2.1`);
            Logger.info('Orchestrator', `Auto-Follow: ${CONFIG.FLAGS.ENABLE_AUTO_FOLLOW ? 'ENABLED' : 'DISABLED'}`);
            Logger.info('Orchestrator', `Auto-Login: ${CONFIG.FLAGS.ENABLE_AUTO_LOGIN ? 'ENABLED' : 'DISABLED'}`);
            Logger.info('Orchestrator', `Dry Run: ${CONFIG.FLAGS.DRY_RUN ? 'ENABLED' : 'DISABLED'}`);

            // Check username availability
            const storedUsername = GM_getValue(CONFIG.LOGIN.USERNAME_STORAGE_KEY, '');
            const fallbackUsername = window.name;
            const username = storedUsername || fallbackUsername;

            if (username) {
                Logger.info('Orchestrator', `Username: ${username}`);
                if (!storedUsername && fallbackUsername) {
                    Logger.info('Orchestrator', 'Note: Username found in window.name, will be migrated to GM storage on first use');
                }
            } else {
                Logger.warn('Orchestrator', '⚠️ ユーザ名が設定されていません');
                Logger.info('Orchestrator', 'ヒント: コンソールで setXCredentials("username", "password") を実行してください');
            }

            Logger.info('Orchestrator', '='.repeat(60));

            // Initial page processing
            await DOMUtils.wait(CONFIG.DETECTION.PAGE_STATE_CHECK_DELAY);
            await this.processCurrentPage();

            // Start monitoring
            this.monitorUrlChanges();
            this.startLogoutMonitoring();

            Logger.info('Orchestrator', 'All systems operational');
        },

        /**
         * Stop the orchestrator
         */
        stop() {
            Logger.error('Orchestrator', 'Orchestrator stopped due to errors');
            // Could implement cleanup here
        }
    };

    // ============================================================================
    // GLOBAL HELPER FUNCTIONS (accessible from browser console)
    // ============================================================================

    // Expose helper functions to unsafeWindow for easy access from console
    // (Tampermonkey uses sandbox, so we need unsafeWindow to expose to page context)
    unsafeWindow.setXCredentials = function (username, password) {
        GM_setValue(CONFIG.LOGIN.USERNAME_STORAGE_KEY, username);
        GM_setValue(CONFIG.LOGIN.PASSWORD_STORAGE_KEY, password);
        console.log('✅ 認証情報を保存しました');
        console.log('Username:', username);
        console.log('Password:', '***' + password.slice(-4));
        console.log('ページをリロードしてください');
    };

    unsafeWindow.setXUsername = function (username) {
        GM_setValue(CONFIG.LOGIN.USERNAME_STORAGE_KEY, username);
        console.log('✅ ユーザー名を保存しました:', username);
    };

    unsafeWindow.setXPassword = function (password) {
        GM_setValue(CONFIG.LOGIN.PASSWORD_STORAGE_KEY, password);
        console.log('✅ パスワードを保存しました');
    };

    unsafeWindow.getXCredentials = function () {
        const username = GM_getValue(CONFIG.LOGIN.USERNAME_STORAGE_KEY, '');
        const password = GM_getValue(CONFIG.LOGIN.PASSWORD_STORAGE_KEY, '');
        console.log('Username:', username || '(未設定)');
        console.log('Password:', password ? '***' + password.slice(-4) : '(未設定)');
        return { username, password };
    };

    unsafeWindow.clearXCredentials = function () {
        GM_deleteValue(CONFIG.LOGIN.USERNAME_STORAGE_KEY);
        GM_deleteValue(CONFIG.LOGIN.PASSWORD_STORAGE_KEY);
        console.log('✅ 認証情報を削除しました');
    };

    // ============================================================================
    // INITIALIZATION
    // ============================================================================
    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', () => Orchestrator.init());
    } else {
        Orchestrator.init();
    }

    // Show helper message on script load
    console.log('%c[X Auto-Follow+Login] ヘルパー関数が利用可能です', 'color: #1DA1F2; font-weight: bold');
    console.log('認証情報を設定: setXCredentials("username", "password")');
    console.log('確認: getXCredentials()');
    console.log('削除: clearXCredentials()');

})();
