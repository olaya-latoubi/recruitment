Q2.1 — Generalization failure
The first thing I'd suspect The training data came from one or two clubs with a specific camera setup. That's it. The model didn't learn to detect hits it learned to detect hits from that angle, at that height, under those lighting conditions.
I'd start by tagging the failures: camera angle, indoor/outdoor, lighting. If they cluster, you know exactly where the gap is. Then I'd run pose estimation alone on the new footage before even looking at hit detection. If keypoints are already broken, everything downstream is broken for that reason ( fix that first)).

Q2.2 — Smash detection
More smash data helps, but only if it actually covers the conditions where the model fails. A hundred new smash clips from the same club won't fix failures at other clubs.
Augmentation on existing clips is cheap and worth doing — flips, speed changes, brightness shifts. It won't manufacture new motion patterns but it reduces sensitivity to superficial variation. Oversampling during training is the simplest lever, though if your smash clips are all from one player you're overfitting to that player's biomechanics, not smashes in general. Loss reweighting is an option but it tends to hurt precision on common hits — test it with that tradeoff in mind.
To verify it actually worked: hold a smash test set from unseen clubs, touch it only at the end. If validation improves but that set doesn't, it's overfitting.

Q2.3 — Which model
Depends entirely on whether hit timing matters to the product.
Jitter corrupts timestamps. If the product surfaces hit timing to users, the jittery model is a problem regardless of its accuracy. If it only classifies hit type and a few frames of error is acceptable, jitter matters much less.
I'd measure frame-to-frame keypoint variance during static moments and timestamp error against manually labeled clips before making a call. Then test both on unseen clubs — stability under new conditions is worth knowing before shipping either.

Q2.4 — Model choice
MoveNet Lightning or YOLOv8-Pose Small. Both run on modest hardware, both are candidates for near-real-time deployment, depending on the hardware.
MoveNet is single-person so you need a detection step upstream for doubles. YOLOv8-Pose handles both in one pass which is cleaner. I'd lean toward YOLOv8-Pose Small for that reason.
Heavier models aren't the answer here. The problems identified above are data and training problems — a bigger backbone on bad data is still a bad model.
