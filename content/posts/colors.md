+++
title = 'Understanding Color: A Programmer's Guide to Light, Perception, and Application'
date = 2024-11-11T10:20:36-05:00
draft = true
+++

I remember the first time I worked with color spaces. It was for an image colorization project, and I had to transform between different color spaces. At first, I thought color was just about those familiar six-digit hex numbers. But as I dived deeper, I realized that color is much more complex. Every time I revisited color theory, I gained a deeper understanding. Concepts that seemed straightforward revealed unexpected depth, and things I once thought were identical turned out to be distinct. This series is my attempt to share what I've learned and help others navigate the fascinating and intricate world of color.

Color is more than just numbers, it's a story of light, perception, and physics. We will explore how light creates color, how our eyes interpret it, and how we can use it in programming. If you’ve ever wondered why different rendering engines treat color differently, or why certain color encodings look so odd when improperly used, then this series is for you. Understanding color can make a big difference when interacting with color systems in practical scenarios, such as calling a rendering API or working with color encodings. I hope to help you represent, manipulate, and work with colors in a way that makes your interactions with graphics libraries more intuitive and informed.

## Do you really know what color is?

We often hear about color in different ways, and sometimes these explanations seem to contradict each other. One common explanation is that color is simply light with different wavelengths. Another explanation is based on our biology: human eyes have three different types of cones for sensing colors, while other animals, like the mantis shrimp, have many more. The great scientist Isaac Newton demonstrated that white sunlight contains all colors by splitting it with a prism into a rainbow. Yet, there are colors, such as magenta, that don't appear in the rainbow at all. So why are there so many different ways to describe something that seems so simple? Are they wrong, or are they just looking at different aspects of color?

The truth is, color is much more complicated than it first appears. It’s not just about seeing red, blue, or green. Color is the result of complex interactions between physical light, the biology of our eyes, and the way our brains interpret signals. Understanding color means understanding the physics of light, the biology of vision, and the perception processes happening in our minds.

## What Is Light, Physically?

To understand color, you first start by skipping anything related to our human body, and focus on what light physically is. In short, light is an electromagnetic wave — essentially, a traveling fluctuation of electric and magnetic fields that moves through space. Maxwell's equation tells us that changing electric field creates magnetic field, and changing magnetic field creates electric field. Therefore the two fields generate each other and propagate through space as electromagnetic waves.

A fundamental property of light is its frequency. It describes how many times the electric and magnetic fields oscillate per second. The SI unit of frequency is Hertz, which is the number of oscillations per second.

The colors we perceive depend on the wavelengths that make up this light. For example, light with a wavelength around 700 nanometers appears red, while shorter wavelengths, like 450 nanometers, appear blue. Most of the colors we see in everyday life come from a mix of many different wavelengths of light combined together.

When we talk about light, it's often more useful to think about it in terms of its frequency domain rather than its time domain. This means we're interested in understanding the contribution of each wavelength to the overall light. To do this, we can apply a mathematical process called the Fourier transform to break down the time-domain representation of light into its frequency components. This is similar to separating the colors of white light with a prism, but mathematically, it gives us a way to see how much of each wavelength is present in a given light source. When we talk about color, we're really talking about the strength of each wavelength in that mix.

Light that consists of just one wavelength is called monochromatic light, which produces a "pure" color, like the bright red of a laser pointer. However, true monochromatic light doesn’t exist naturally. Even lasers that seem to produce a single color still have a small range of wavelengths. This range is narrow enough that we perceive it as a single, pure color. Most natural light sources, such as the sun or a light bulb, emit a wide range of wavelengths, resulting in the complex mix of colors we perceive in daily life.

## Additive Color Mixing

The process by which different wavelengths of light mix is known as additive color mixing. When multiple light sources with different wavelengths come together, the resulting light is a combination of all those wavelengths. In additive color mixing, each light source contributes to the overall spectrum, creating a color that contains the combined characteristics of the original lights. For example, if you mix together light of different wavelengths, their energy combines, and the color you perceive is a result of that mixture.

This is different from subtractive mixing, which happens with pigments or paints, where colors mix by absorbing (subtracting) wavelengths. Additive color mixing is important to understand because it’s how colors are created in many displays and lighting technologies. By controlling the strength of each wavelength, we can create a wide range of colors from just a few basic components.

Color, then, isn't simply an inherent property of light. It's an interaction involving physics, biology, and even the unique workings of our brains.

