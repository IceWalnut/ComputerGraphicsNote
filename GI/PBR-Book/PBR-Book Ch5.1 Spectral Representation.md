# PBR-Book Ch5 Color and Radiometry

[Color and Radiometry (pbr-book.org)](https://www.pbr-book.org/3ed-2018/Color_and_Radiometry)

In orderto precisely describe how light is represented and sampled to compute images, we must first establish some background in *radiometry*-- the study of the propagation of eletromagnetic radiation between approximately 380 nm and 780  nm, which account for light visible to humans. The lower wavelengths ($\lambda \approx 400  \ \mathrm{nm}$)  are the bluish colors, the middle wavelengths ($\lambda \approx 550 \ \mathrm {nm}$) are the greens, and the upper wavelengths ($\lambda \approx 650 \ \mathrm {nm}$) are the reds.

In this chapter, we will introduce four key quantities that describe electromagnetic radiation: flux, intensity, irradiance, and radiance. These radiometric quantities are each described by their ***spectral power distribution (SPD)***—a distribution function of wavelength that describes the amount of light at each wavelength. The [`Spectrum`](https://www.pbr-book.org/3ed-2018/Color_and_Radiometry/Spectral_Representation.html#Spectrum) class, which is defined in Section 5.1, is used to represent SPDs in **pbrt**.





# PBR-Book Ch5.1 Spectral Representation









[Spectral Representation (pbr-book.org)](https://www.pbr-book.org/3ed-2018/Color_and_Radiometry/Spectral_Representation)