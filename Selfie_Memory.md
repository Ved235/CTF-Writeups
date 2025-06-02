
# Selfie Memory (N0PSctf 2025)

In this challenge, we have an image `noopsy_selfie.png`, taken by “Joseph” in 1807. The hint says:

> “You have something, but can it be transformed?”

On some 'careful reading' reading of the challenge description we can identify the clue that we have to perform a Fourier transform (Joseph Fourier)

Now there are 2 methods to solve this chall once you have identified this:
1. The classic traditional method (good for learning)
2. Using a hack?

## The traditional method
### 1. File Inspection

It is always a good practice to check the metadata of the file in case there are any additional hints or clues.
In this case the metadata didn't reveal anything interesting. So, we can start with the Fourier transform.
![image](https://github.com/user-attachments/assets/207cb48a-5865-4995-8c54-1ad581d2001c)

   
    



### 2.  Computing the 2D Fourier Transform

We’ll use Python with NumPy and Matplotlib (and Pillow) to load the image, convert it to grayscale and compute its 2D Fourier transform.

```python
import numpy as np
from PIL import Image
import matplotlib.pyplot as plt

# 1)Load the PNG and convert to grayscale
img = Image.open('noopsy_selfie.png').convert('L')
arr = np.array(img, dtype=np.float32)

# 2)Compute 2D FFT and shift zero-frequency to center
F = np.fft.fft2(arr)
F_shifted = np.fft.fftshift(F)
magnitude_spectrum = 20 * np.log10(np.abs(F_shifted) + 1)

# 3)Plot the transformed image
plt.imshow(magnitude_spectrum, cmap='gray')
plt.axis('off')
plt.title('noopsy_selfie.png')
plt.show()
```
Converting to grayscale focuses on intensity channel, which carries the embedded information in the Fourier domain. Since the raw FFT values span many orders of magnitude, we take a log scale to visualize
![image](https://github.com/user-attachments/assets/44c7120e-e306-420b-a65c-6051f6624a4f)


## The hack method
Upload the challenge to **ChatGPT** 💀.  Prompt Engineering required obviously *~~(joking)~~*
```
Solve this CTF challenge:

At the end of this lovely moment, n00psy gives you a picture of him. "I will never forget this moment, thank you for coming with me!", he says. "Also, sorry if the quality of this picture is a bit poor on the edges, this is an old picture that my friend Joseph took in 1807."

Hint:

You have something, but can it be transformed?
```

![ezgif-60718d57511ef0](https://github.com/user-attachments/assets/ff4c499d-972b-49a5-8fd9-cfd22562dc15)




## Final Flag
The challenge did require a new method which isn't a very common approach, but the hint might have made it a bit too easy. (*it's not like i am complaining)*
```
N0PS{i_love_fourier}
```



