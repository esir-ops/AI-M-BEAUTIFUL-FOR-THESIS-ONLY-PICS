# Glasses model slot (empty)

Drop a Google Teachable Machine TensorFlow.js export here to replace the
pixel-heuristic glasses detector:

```
models/glasses/model.json
models/glasses/metadata.json
models/glasses/weights.bin
```

The app loads it automatically on next refresh (console logs `[tm:glasses] model ready`).
While this folder has no `model.json`, the built-in heuristic runs instead.

See `../HOW-TO-TRAIN.md` for the full training walkthrough. One class name must
contain the word **`glasses`**.
