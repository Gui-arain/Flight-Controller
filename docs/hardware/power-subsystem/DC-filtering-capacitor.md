# DC filtering with capacitors

Capacitors are used throughout the whole power distribution system to ensure smooth voltage levels of power rails.
However these needs to be properly sized.

## DC capacitance bias

When using MLCC capacitors in DC mode (which is the case for DC voltage filters), the real value capcitive values are significantly dependant on the DC voltage applied.

Let's take the CL05A105KA5NQN 1uF 25Vdc 0402 capacitor  for example:
[https://weblib.samsungsem.com/mlcc/mlcc-ec-data-sheet.do?partNumber=CL05A105KA5NQN]

When operating at 5V we get -60.86% reduced effective capacitance according to the DC Bias Characteristics.
So this 1uF capacitors only truly behaves like a 391.4 nF capacitor at this voltage.

This bias characteristics is generally impacted by:
- The voltage rating: a bigger volatge rating capacitor will have less bias at a same DC voltage
- The package size: a bigger package capacitor will have less bias at a same DC voltage

**Here are some key values calculated:**

5V DC:

| Name             |  Nominal cap | True cap    |   Rated Vdc   |   package     |   TCC     |   tolerance   |
| ---------------- | ------------ | ----------- | ------------- | ------------- | --------- | ------------- |
| CL05B104KO5NNN   |  100nF       | 90.5nF      |   16Vdc       |   0402        |   X7R     |   ±10%        |
| CL05A105KA5NQN   |  1uF         | 391.4nF     |   25Vdc       |   0402        |   X5R     |   ±10%        |
| CL10A105KB8NNN   |  1uF         | 656nF       |   50Vdc       |   0603        |   X5R     |   ±10%        |
| CL10A225KO8NNN   |  2.2uF       | 965.8nF     |   16Vdc       |   0603        |   X5R     |   ±10%        |
| CL10A106MA8NRN   |  10uF        | 4.72uF      |   25Vdc       |   0603        |   X5R     |   ±20%        |
| CL21A106KAYNNN   |  10uF        | 4.97uF      |   25Vdc       |   0805        |   X5R     |   ±10%        |
| CL21A226MAQNNN   |  22uF        | 10.71uF     |   25Vdc       |   0805        |   X5R     |   ±20%        |