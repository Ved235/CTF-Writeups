# The Weather Oracle (07CTF 2025)

> The year is 20XX. Meteorological agencies worldwide receive an anonymous file, containing disturbing weather data. A cataclysmic storm is forming on an unprecedented scale. Forecast models show an impossible convergence of hurricanes, tornadoes, and heat waves all in a single location. But something doesn’t add up. A rogue scientist, now missing, left a hidden message within the data. He claimed that this "perfect storm" is not natural. Before vanishing, he uncovered a classified climate manipulation project. The only clue he managed to transmit is buried within this file. We need you to undisclose what was hidden.

We were provided with a GRIB file which is the "standardized format used for storing and distributing meteorological data". The challenge basically wanted us to find a hidden encoded flag in this file.

***

## Step 1: Try to open and view this file

I have never worked with this type of file before, so I asked ChatGPT to suggest some tools to open this file. It suggested **wrgib2,** but I was really lazy to build a tool from source especially on my mac so I opted for a GUI tool called **panoply**.&#x20;

