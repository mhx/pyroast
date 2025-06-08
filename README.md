# pyroast

A very simple (and largely vibe-coded), but surprisingly helpful script
to train models to predict weight loss, roast color, dose and grind
settings from green coffee bean properties and roast settings.

While the script is tailored around my particular setup for roasting and
preparing coffee, it should be fairly easy to adapt.

The main objective of this script is to reduce the number of sub-optimal
roasts and significantly cutting down the time to dial in the grinder
setting after roasting.

## Dataset

The input dataset in my case is a spreadsheet that logs every roast as
a separate row using the following columns that are relevant for model
training/prediction:

- `Altitude`: The altitude at which the coffee was grown. Can be a
  range, in which case the average is used as a predictor.

- `Density`: The bulk density of the green coffee in grams per liter.
  I'm measuring this using a 250ml lab cylinder that is vibrated until
  the beans don't compress anymore. The weight is then multiplied by 4.

- `Weight`: The average bean weight in milligrams, determined by weighing
  50 "average" beans.

- `Moisture`: The green bean moisture content in percent. Can be measured
  using a moisture meter or taken from the supplier.

- `Profile` and `Level`: The profile and level used for roasting. I'm
  using a Kaffelogic nano 7e and I'm generally sticking to the "Washed"
  and "Natural" profiles. Most of my roasts are quite light, usually
  between 105 and 120 on the Tonino scale.

- `Loss`: Weight loss during roasting in percent. This is the target for
  the loss model. It is *also* used as a predictor for the dose and grind
  models.

- `Color`: Color of the ground coffee on the Tonino scale. This is the
  target for the color model.

- `Dose`: Dose for a double-shot espresso. This is the dose that for me
  will neatly fit into an 18g VST basket. Target for the dose model.

- `Grind`: This is the setting on my Titus grinder, which has a scale
  labeled in degrees. Usually, the range for espresso is between 10 and
  60 degrees. Target for the grind model.

There's a bunch of other columns in the spreadsheet that are (currently)
not used.

## Model Training and Evaluation

Training the models is pretty straightforward:

```
$ ./pyroast train --tsv coffeeroasting.tsv
• loss   using  69/71 rows
• color  using  49/71 rows
• dose   using  52/71 rows
• grind  using  52/71 rows
✅ models written to models
```

Evaluation is equally simple:

```
$ ./pyroast evaluate --tsv coffeeroasting.tsv
Loss  | RMSE: 0.46 ± 0.03 (69/71 rows)
Color | RMSE: 3.67 ± 1.01 (49/71 rows)
Dose  | RMSE: 0.45 ± 0.06 (52/71 rows)
Grind | MAE : 5.38 ± 1.01 (52/71 rows)
```

## Use Case: Determining Roast Level

Since I'm only roasting about 1.5kg of coffee per month, I'm not very
good at looking at a green bean and knowing exactly what roast level
to pick to get to a desired roast degree (i.e. color). I can usually
guess if this should be roasted on the lighter or darker side, but I
can't easily translate that into a roast level. Also, I usually only
have between 500g and 2kg of each bean, and a lot of them are quite
"vintage" (from around 2015/2016). All in all, it's too many variables
for me to keep track of in my head.

So, when I want to roast a new batch of beans for the first time, I
can simply use:

```
$ ./pyroast level --profile 'KL Washed' --moisture 11.3 --density 687 \
            --weight 168 --bean-age 9 --altitude 1650 --target-color 115
Best roast level for target color 115 is: 1.0 (error: 0.12)
```

I can also predict the color and weight loss for that level:

```
$ ./pyroast predict-row --profile 'KL Washed' --moisture 11.3 --density 687 \
            --weight 168 --bean-age 9 --altitude 1650 --level 1.0
Loss  : 12.60
Color : 115.12
```

Turns out that after roasting the beans at exactly that level, the
actual loss was 13.3% instead of 12.6%. However, the color was spot
on and measured as 115.

## Use Case: Determining Grind Setting

When roasting batches as small as 80g, you only get a limited number
of shots to dial in the grind setting. If you're off by 10 degrees,
the shot quality is usually unacceptable (i.e. instead of a 27 second
shot, which I aim for on average, you're either at less than 20 seconds
or more than 40 seconds). I've got *some* feeling for picking a good
setting for a new roast, but sometimes I'm just way off, and nailing
it on the first shot is actually quite rare.

My experience with the dose and grind models is quite limited at the
moment, but so far I'm honestly surprised that in about 4 out of 5
cases, the suggested setting was absolutely spot on:

```
$ ./pyroast predict-row --profile 'KL Washed' --moisture 11.3 --density 687 \
            --weight 168 --level 1.0 --bean-age 9 --loss 13.3 --altitude 1650
Loss  : 12.60
Color : 115.12
Dose [g]: 16.97
Grind [°Titus]: 32.64
```

I've ground the beans on that setting, using a 17g dose, and got a
40g shot in 27 seconds. As mentioned before, the color measured as
exactly 115.
