### Performance related information

I mainly use emacs for C++ development and viewing/editing log files, so you can expect to find performance related modifications in these areas.

I created [fast-emacs](https://github.com/geza-herman/emacs/tree/fast-emacs) branch (it is based on Andrea Corallo's `feature/native-comp` branch), which contains these small modifications so far:
 - Process IO modification: together with IO-related settings described below, on my computer `shell` can cat large files 60x faster than original emacs.  Working with large log files is much easier this way.  Emacs is still slower than fast terminals, but its speed is much more acceptable.
 - Rendering optimization: face lookup by face name is done by hash table instead of linear search.  This makes display rendering much faster in certain cases (here's such a case: when I had the same buffer open in several windows, and in evil mode, I switched to visual mode to select some text, emacs slowed down to ~2 fps. With this modification, slowdown is almost unnoticable).
 - Disable composition.  As I don't use ligatures nor fonts which need composition, I disabled this feature.  If you'd like to still use composition, reverse the related commit (it is just 1 line).

Note: these modifications are not tested by a wider audience. They work for me, but maybe they will cause some problems for you. I just shared these modifications because maybe they can be helpful for others (I bet that an expert emacs developer would do things differently).

Also, I use these settings related to performance:

```lisp
;; Don't care about bidirectional text. These settings make processing long lines faster.
(setq bidi-inhibit-bpa t)
(setq-default bidi-paragraph-direction 'left-to-right)

;; IO-related tunings
(setq process-adaptive-read-buffering nil)
(setq read-process-output-max (* 1024 1024))

;; C++ level 3 decoration can be very slow in certain occasions.  As I use lsp with semantic
;; highlighting, emacs's decoration doesn't matter too much
(setq font-lock-maximum-decoration '((c-mode . 2) (c++-mode . 2) (t . t)))

;; I use smartparens which can highlight matching pairs, so I don't need
;; emacs's default blink parenthesis functionality
(setq blink-paren-function nil)

;; With original setting, helm calls constant (and unnecessary) forced mode-line updates
(setq helm-ff-keep-cached-candidates nil)
```

I use this pager (derived from Mikael Sjödin's pager), because emacs's pager can be slow in certain cases (and I like the behavior of this pager better than the original one):
```lisp
(setq-default my-pager-column-goal 0)
(make-variable-buffer-local 'my-pager-column-goal)

(defun my-pager-store-column ()
  (if (not (memq last-command '(my-pager-page-down my-pager-page-up my-pager-row-up my-pager-row-down)))
      (setq my-pager-column-goal (current-column))))

(defun my-pager-restore-column ()
  (move-to-column my-pager-column-goal))

(defun my-line-move (lines)
  (unless (line-move-1 lines t)
    (if (> lines 0)
        (goto-char (point-max))
        (goto-char (point-min)))))

(defun my-pager-scroll-screen (lines)
  (save-excursion
    (goto-char (window-start))
    (my-line-move lines)
    (set-window-start (selected-window) (point)))
  (my-line-move lines))

(defun my-pager-page-down ()
  (interactive)
  (my-pager-store-column)
  (if (pos-visible-in-window-p (point-max))
      (goto-char (point-max))
      (my-pager-scroll-screen (- (1- (window-height))
                                 next-screen-context-lines)))
  (my-pager-restore-column))

(defun my-pager-page-up ()
  (interactive)
  (my-pager-store-column)
  (if (pos-visible-in-window-p (point-min))
      (goto-char (point-min))
      (my-pager-scroll-screen (- next-screen-context-lines
                                 (1- (window-height))))
      (my-pager-restore-column)))

(defun my-pager-row-up ()
  (interactive)
  (my-pager-store-column)
  (save-excursion
    (goto-char (window-start))
    (my-line-move -1)
    (set-window-start (selected-window) (point)))
  (while (save-excursion
           (my-line-move (+ scroll-margin 2))
           (>= (point) (window-end)))
    (my-line-move -1))
  (my-pager-restore-column))

(defun my-pager-row-down ()
  (interactive)
  (my-pager-store-column)
  (save-excursion
    (goto-char (window-start))
    (my-line-move 1)
    (set-window-start (selected-window) (point)))
  (while (save-excursion
           (my-line-move (- 0 scroll-margin))
           (< (point) (window-start)))
    (my-line-move 1))
  (my-pager-restore-column))

(global-set-key (kbd "<next>")'my-pager-page-down)
(global-set-key (kbd "<prior>") 'my-pager-page-up)
(global-set-key (kbd "<S-up>") 'my-pager-row-up)
(global-set-key (kbd "<S-down>") 'my-pager-row-down)
```

For faster startup, I use these in `early-init.el`:
```lisp
(setq package-quickstart t)

;; remove searching for .gz files, as all my packages are uncompressed
;; this halves how many files emacs tries to open (if you have a lot of
;; packages, emacs executes a lot of failing file opens)
(setq jka-compr-load-suffixes nil)
(jka-compr-update)

;; set file-name-handler-alist to nil, to avoid a lot of regexp searches
;; this variable will be restored in init.el
(setq my-file-name-handler-alist-orig file-name-handler-alist)
(setq file-name-handler-alist nil)

;; also set file-name-handler-alist to nil during 'require'
;; this is needed because of deferred package loading
(advice-add 'require :around (lambda (orig-fun &rest args)
  (let ((file-name-handler-alist))
    (apply orig-fun args))
))
```

and I put this at the beginning of `init.el`:
```lisp
;; restore file-name-handler-alist.  I restore this variable at the beginning,
;; as some package may add something to the list.
(setq file-name-handler-alist my-file-name-handler-alist-orig)
```
With these settings, startup time decreased from 2.5 sec to 1.2 sec (tested with loading a C++ file with all needed packages are loaded, like `lsp`, etc.). `emacs -kill` runs in 0.4 seconds.

### How to decrease line spacing

As of now (Jan 2020), Emacs doesn't support negative `line-spacing`.  However, I wanted to have more lines on my screen.  How to do this?  I found an ugly solution.  It involves two steps:
 - Modify the font: I used fontforge to decrease ascent and descent.  Open the font in fontforge, go to Element/Font Info/OS/2/Metrics, and decrease `Win Ascent` and `Win Descent`.  I also decreased `Typo Ascent`/`Descent` and `HHead Ascent`/`Descent` by the same amount (note that these two descent values are usually the inverse of `Win Descent`). For example, I use Roboto Mono.  The original values were 2146 and 555.  I decreased them to 1821 and 430.  I also modified the glyphs of `(` and `)` to be a little bit smaller.
 - As the modified font has a smaller height, neighboring lines may overlap. This makes emacs rendering slower, because (as far as I understand) it handles overlapping by rendering overlapped lines more than once. However, I turned this special overlap handling off, and I didn't notice any glyph clipping, so I'm good without this extra overlapping rendering, rendering speed is back to normal.  You can find this little tweak in my [my-modifications](https://github.com/geza-herman/emacs/tree/my-modifications) branch.
