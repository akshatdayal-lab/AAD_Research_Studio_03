# Rebar check

Checks a rebar mat before the pour. For every bar it answers two questions: is it the right size, and is it in the right place. Each bar is labelled green, yellow or red and shown in AR on an iPhone.

| Label | Meaning |
|---|---|
| green | right size, right place |
| yellow | right size, but off by more than the tolerance (up to twice the tolerance) |
| red | wrong size, wrong place, or missing |

Part of Concrete Evidence, IAAC MaAI, Research Studio Smart Construction.

## How it works

```
1 Build the AI      dataset > train YOLO in Colab > best.pt > test on your photos
2 Measure           top down photo + 4 ArUco markers > size and spacing of every bar in mm
3 Scan and compare  Polycam scan > line up with the design (markers, then ICP) > match bars
4 Show it in AR     coloured bars > USDZ > AR Quick Look on iPhone or iPad > fix and rescan
```

The AI finds the bars. Everything after that is geometry. Each part can be tested on its own with simulated data, so nothing waits for the rest.

## Folder

```
config/test_mat.json          the design: bars, spacing, diameter, tolerance, marker positions
notebooks/train_rebar_yolo.ipynb   steps 1 to 3, runs in Google Colab
rebar_check/                  the code (measure2d, align, match, export_usdz, report)
scripts/measure_photo.py      photo > size and spacing per bar
scripts/run_check.py          scan > placement per bar > report and USDZ
scripts/simulate_photo.py     fake photo with known mistakes, for testing
scripts/simulate_scan.py      fake scan with known mistakes, for testing
```

## Setup on your laptop

Python 3.10 or 3.11 (Open3D is not on the newest Python yet).

```
pip install -r requirements.txt
```

## Try it now on simulated data

The test mat has five planted mistakes: X03 is 18 mm off (yellow), Y04 is 32 mm off (red), Y02 is a 10 mm bar instead of 12, X05 is a 16 mm bar instead of 12, and Y06 is missing.

```
python scripts/simulate_photo.py
python scripts/measure_photo.py --mat config/test_mat.json --photo photos/sim/mat.jpg --detections photos/sim/detections.json

python scripts/simulate_scan.py
python scripts/run_check.py --mat config/test_mat.json --scan scans/simulated/scan.ply --corners scans/simulated/corners.json --sizes results/photo/report.json --out results/simulated
```

Both steps should find all five mistakes. Open `results/simulated/rebar_check.usdz` on an iPhone to see it in AR.

## The real steps

**Step 1, get the data.** On Roboflow Universe, search "rebar" and pick a dataset with segmentation (outlines), because bar thickness can't be measured from boxes. Download it in YOLOv8 format.

**Step 2, train.** Open `notebooks/train_rebar_yolo.ipynb` in Google Colab, set the runtime to T4 GPU and run it. It saves `best.pt`.

**Step 3, test.** Run the last cells of the notebook on 20 of your own phone photos. If the outlines sit on the bars, this step is done.

**Step 3b, measure.** Build the test mat as in `config/test_mat.json` (edit the file if you build it differently). Print four ArUco markers (DICT_4X4_50, ids 0 to 3, 100 mm wide, which you can generate at chev.me/arucogen) and place them just outside the mat corners, at the positions in the config. Take one photo from straight above, with all four markers in frame.

```
python scripts/measure_photo.py --mat config/test_mat.json --photo photos/mat.jpg --model best.pt
```

**Step 4, GitHub.** Create a repo and push this folder. The `.gitignore` already keeps datasets, `best.pt`, scans and results out.

**Step 5, scan.** In Polycam, scan the mat with the markers in view and export as PLY or OBJ. Then note the marker centres, or click them with `--pick` in the next step.

**Step 6, compare.**

```
python scripts/run_check.py --mat config/test_mat.json --scan scans/mat.ply --pick --sizes results/photo/report.json
```

Add `--allow-scale` if you scanned in Photo mode instead of LiDAR mode.

**Step 7, AR.** AirDrop `results/rebar_check.usdz` to the iPhone and tap it. Fix the red and yellow bars, then photograph and scan again until every bar is green.

## Limits to know before the presentation

- **Thickness comes from the photo, not the scan.** iPhone LiDAR depth is only good to roughly a centimetre, so it can't tell a 10 mm bar from a 12 mm one. The photo can, if it is sharp and close enough. From about 1 m above, a 12 MP phone gives roughly 0.35 mm per pixel, so a 12 mm bar is about 35 pixels wide. That's enough to tell 10 from 12, but not from much higher up.
- **Height matters a little.** The markers sit on the board and the bars sit higher, so a photo that isn't straight down shifts the bars slightly. Shoot from straight above, or raise the markers to the height of the top bars.
- **Crossing bars hide each other.** Where bars cross, the code skips that stretch instead of guessing.
- **The tolerance is a setting.** `placement_tolerance_mm` in the config is 10 mm by default. Set it to the value in the standard or the drawings you are checking against.
- **The simulated tests prove the geometry, not the AI.** How well real photos work depends on the trained model.
