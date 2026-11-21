# Two-stage detection: CFAR candidates + CNN classifier (pre-registered plan)

Status: **plan, committed before any chip was extracted or any model was trained.** Changes after this commit are limited to bug fixes; every other change is listed under "Deviations" with its reason.

## Question

Does a small CNN that classifies CFAR candidates improve vessel detection over the best single-stage CFAR detector, on the same 49 xView3-SAR validation scenes and under the same scoring rules as the 49-scene benchmark?

Baseline to beat (from `benchmark/cfar_xview3_multiscene.ipynb`): **Hollow OS-CFAR at the locked T = 8.1 dB, pooled F1 70.7%** (precision 74.0%, recall 67.7%, 1,642 evaluated vessels).

## 1. Candidate generator (stage 1)

- Hollow OS-CFAR, guard 9 × 9, outer 21 × 21, 85th percentile, **T = 5.0 dB** (the locked low reference threshold of the benchmark), brightness floor −30 dB, same preprocessing, window, water mask, closing and minimum area as the benchmark.
- On the 49 scenes this gives recall 89.6% with 1,621 false alarms. Vessels it misses cannot be recovered by stage 2, so **89.6% is the recall ceiling** of the two-stage system.

## 2. Chips and labels

- One chip per candidate: **64 × 64 pixels (640 m × 640 m)** centred on the candidate centroid, **two channels: VH and VV in dB**, clipped to [−35, 5] dB and scaled to [0, 1]. Chips crossing the window edge are padded with −35 dB.
- Label from the benchmark matching (one-to-one Hungarian, 20 px):
  - **positive (1):** candidate matched to an evaluated vessel (`is_vessel == True`);
  - **negative (0):** candidate matched to nothing and not within 20 px of a labelled non-vessel or unknown object;
  - **excluded from training:** candidates within 20 px of a labelled non-vessel or unknown object (these are "ignore" in scoring, so they are neither rewarded nor penalised there either). They are still classified at test time and scored with the ignore rule.
- Stored per chip: scene id, candidate position, label, and for positives the matched vessel's `source` and `confidence`.
- Known label noise: some negatives may be real but unlabelled vessels (seen in the two-scene benchmark). This is not corrected.

## 3. Split: scene-grouped 5-fold cross-validation

- The 49 scenes are shuffled once with seed 42 and split into 5 folds of 9-10 scenes. **No scene appears in both training and test of the same fold.**
- Each fold's model is trained on the other 4 folds and predicts its own fold. Predictions from the 5 test folds are combined, so every candidate of every scene gets exactly one out-of-fold prediction. This allows scoring on all 49 scenes, directly comparable with the baseline.
- Within each training set, 1 of the 4 training folds is held out for early stopping only.

## 4. Model and training (fixed before training)

- Small CNN, trained from scratch: 4 blocks of [conv 3 × 3 → batch norm → ReLU → max pool 2 × 2] with 32, 64, 128, 128 filters, global average pooling, dropout 0.3, one output (vessel probability). About 0.3 M parameters.
- Loss: binary cross-entropy with positive-class weight = (negatives / positives) of the training set.
- Adam, learning rate 1e-3, batch size 64, at most 40 epochs, early stopping on validation loss with patience 5 (validation = the held-out training fold).
- Augmentation (training only): random horizontal and vertical flips and 90° rotations.
- Seed 42 for everything that is random. One training run per fold; no hyperparameter search.

## 5. Decision rule and scoring

- **Primary:** a candidate is kept as a detection if its out-of-fold probability is **≥ 0.5**.
- Kept candidates are scored per scene with exactly the benchmark function (Hungarian matching, 20 px, ignore rules), then pooled over the 49 scenes.
- Reported, all with 95% Wilson intervals for precision and recall:
  - two-stage system (CFAR T = 5.0 + CNN ≥ 0.5): all evaluated vessels and HIGH-confidence labels;
  - baselines recomputed in the same run: OS-CFAR T = 8.1, CA-CFAR T = 11.6, OS-CFAR T = 5.0 alone;
  - **recall by label source** (`ais/manual` vs `manual`) for the baselines and the two-stage system. `manual` labels are vessels found by analysts without an AIS match, the closest proxy for "dark" vessels in this dataset;
  - precision-recall curve of the two-stage system over CNN thresholds (secondary; the best point on it is optimistic).
- **Success criterion:** pooled F1 of the two-stage system minus pooled F1 of OS-CFAR T = 8.1, with a 95% confidence interval from a scene-level bootstrap (1,000 resamples of the 49 scenes, seed 42). The improvement is claimed only if the interval excludes 0. The result is reported whatever it is.

## 6. Notebooks and data handling

- `cnn/extract_chips.ipynb`: downloads each scene (VH, VV, owiMask only), runs the candidate generator, writes chips and labels, deletes the scene. Resumable; chips are backed up to the same single Google Drive folder used by the benchmark (about 50 MB in total). Signed links are never printed or saved.
- `cnn/train_eval.ipynb`: cross-validation training, out-of-fold prediction, scoring and the bootstrap.
- Neither data nor chips are committed to the repository.

## Later stage (not part of this pre-registration)

After this plan is completed, an end-to-end detector (a pretrained YOLO model fine-tuned on small boxes derived from the point labels) will be evaluated on the same 49 scenes with the same scene-grouped folds and scoring, so that single-stage CFAR, CFAR + CNN and end-to-end YOLO can be compared side by side. Its exact settings will be fixed in a separate plan before it is trained. Segmentation models such as SAM are not planned: the labels are points, not outlines, and vessels are only a few pixels long at 10 m resolution.

## Deviations

None yet.
