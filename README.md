# Start here - attendee setup

📘 **[Download the Attendee Guide (PDF)](guides/Attendee-Guide.pdf)** - every module, every lab, and
the reference tables, in one file. Grab it now; you will want it open all day and it is yours to keep.

Everything today runs in a **browser**, on any OS, with no installs. Work through
these steps in order. After each one, check the "Done looks like" line before moving on.

**Budget about 20 minutes for setup.** The last step (building your models) does the
most work, so kick it off and let it run while you settle in.

## 0. Claim your login  (~1 min)
Open the shared allocation sheet (the `aka.ms` link on the welcome slide) and add your
initials next to an unclaimed `userNNNN` row. Sign in to Fabric with that account.
> **Done looks like:** you are signed in and can see your own workspace, plus the shared
> workspace named on the welcome slide.

## 1. Create your own workspace  (~1 min)
Create a **new, empty workspace** of your own. Everything you build today lives there, so
nothing clashes with anyone else. You do not need to make a lakehouse: step 3 does that.
> **Done looks like:** you are in your own new workspace, not the shared one.

## 2. Copy the setup notebook and run it  (~2 min)
Open the **shared** workspace (you have view access), find the `00-setup` notebook, and
**save a copy of it into your own workspace**. Then open your copy and **Run all**. It
imports the lab notebooks from GitHub into a `Labs` folder and shrinks your Spark settings
so the whole room fits on the shared capacity.
> **Done looks like:** the last cell says `Setup complete`, a `Labs` folder appears holding
> `0-create-lab-models`, `lab02-storage-modes`, `lab07-diagnose-slow-visuals` and
> `lab08-prove-the-improvement`, and the output shows `starter maxNodes : 1`.

**No "Save a copy" on the menu?** Do not wait. In **your own** workspace create a new
notebook, set the language to **Python** (not PySpark), paste this in, and run it. It
pulls `00-setup` straight from GitHub, then carry on as above.
```python
%pip install -q semantic-link-labs
import sempy_labs as labs
labs.import_notebook_from_web(
    notebook_name="00-setup",
    url="https://raw.githubusercontent.com/dax-tips/SuccessfulSemanticModelling/main/labs/notebooks/00-setup.ipynb")
```

## 3. Build your lab models  (~10-15 min)
Open `0-create-lab-models` and **Run all**. It creates your `workshop` lakehouse, shortcuts
the shared data into it, and builds your six checkpoint models. This is the long one, so
start it, then grab a coffee.
> **Done looks like:** the final cell lists six semantic models: `01 Star Schema (fixed)`,
> `01 Baseline (messy)`, `03 RLS`, `04 Scaling`, `06 DAX + Calendar`, `07 Slow Visual Triage`.

## 4. Open Lab 1
You are ready. Open `lab01` and begin.

---

## If something stalls
On busy conference wifi a login or a model build can lag. Do not sit blocked:

- The presenter keeps a **reference set of the six models** in the shared workspace. For
  the watch-and-diagnose modules (2, 5, 7, 8) you can follow along on the reference model
  while your own build finishes.
- The hands-on modules (1, 3, 6, 9) need your **own** models. If step 3 is still running,
  leave it going and catch up at the next break. Every lab starts from its own checkpoint,
  so you can always rejoin at the next module.
- If step 3 fails outright, re-run it (it is safe to run again). If it still fails, flag a
  helper rather than losing time debugging alone.
