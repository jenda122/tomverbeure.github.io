---
layout: post
title: A HP 8007B Pulse Generator and Measuring the Length of a Coax Cable
date:   2023-02-06 00:00:00 -0700
categories:
---

* TOC
{:toc}

# Introduction

Living in the Bay Area is not all roses, but one amazing benefit is the amount of electrical engineers
who are retiring, downsizing, and finally getting rid of their old test equipment. There's John,
who has my number on speed dial when he wants to get rid of signal generators, spectrum analyzers, and 
frequency counters. ("I know the price is way too low, but I don't want to sell who'd just put it on 
eBay"), and there's Lew who's similarly cleaning up his lab of old but extremely well maintained RF
equipment.

[![HP 8007B Pulse Generator](/assets/hp8007b/hp8007b.jpg)](/assets/hp8007b/hp8007b.jpg)
*(Click to enlarge)*

A few weeks ago, Lew posted not one but 2 HP 8007B pulse generators on Craiglist for a total of $50,
sold as a pair only. Granted, only one unit was working and the other for parts only, but Lew had 
already tracked down the failure to the output amplifier so it might be fixable. That's maybe for 
another time...

Let's be clear: I have no use for a pulse generator, and before writing this blog post I didn't know 
what these things are normally used for either. 
[Wikipedia](https://en.wikipedia.org/wiki/Pulse_generator) tells me they are often used *to create a 
stimulus to analyze a device under test, confirm proper operation, or pinpoint a fault in a device*.

For $50 it seems like a fun thing to play with, so I made the 2 mile trek from Sunnyvale to 
Mountain View to pick up the units.

# The HP 8007B Pulse Generator

I don't know the exact data when the 8007B came to be, but the manual was printed in October 1972,
makes it more than 50 years old. 

Here's some information I've been able to gather on the web:

An [HP Pulse Generators catalog](/assets/hp8007b/pulsegen_cat70s.pdf) contains a few bullet points
and a pulse generation comparison table:

[![HP 8007B pulse generator in pulse generator catalog](/assets/hp8007b/hp8007b_pulsegen_cat70s.jpg)](/assets/hp8007b/hp8007b_pulsegen_cat70s.jpg)

[![Pulse generator comparison table](/assets/hp8007b/pulsegen_comparison_table.jpg)](/assets/hp8007b/pulsegen_comparison_table.jpg)

For a bit more detail, you can turn to page 287 of the huge 
[1976 HP general products catalog](http://hparchive.com/Catalogs/HP-Catalog-1976.pdf)
which has a nice [one-page spec sheet](/assets/hp8007b/8007b_spec_sheet.pdf).

As usual, Keysight still maintains an [8007B product page](https://www.keysight.com/us/en/product/8007B/pulse-generator.html#resources)
which has the operating and service manual, but the scan quality is not great. I decided to
splurge on a high quality scanned copy at [ArtekManuals](https://artekmanuals.com) for $12.[^1]

[^1]: ArtekManuals is allowed to distribute these manuals under license from Keysight. Purchased copies
      can not be shared with others.

While gathering these documents, I learned about some applications where the 8007B is considered
useful:

* cable length measurement

    ![Cable length measurement setup](/assets/hp8007b/cable_length_measurement.jpg)

* testing the noise immunity of TTL logic
    
    ![TTL noise immunity test](/assets/hp8007b/ttl_noise_immunity_test.jpg)

* measuring the sensitivity of a flip-flop

    ![Measuring FF sensitivity](/assets/hp8007b/measuring_ff_sensitivity.jpg)

* testing the threshold voltage of digital logic

    ![Logic threshold testing](/assets/hp8007b/logic_threshold_testing.jpg)

* time interval measurement

    ![Time interval measurement](/assets/hp8007b/time_interval_measurement.jpg)

Let's go back to the key features:

* 10Hz to 100MHz
* 2ns to 250us transition times
* highly linear slopes
* +-5V output

Of those, a transition time of 2ns is probably the one that is hardest to get with modern signal
generators.

# References

* [Reverse-Engineering an IC to fix an HP 8007A Pulse generator](http://www.dasarodesigns.com/projects/reverse-engineering-an-ic-to-fix-an-hp-8007a-pulse-generator/)
* https://www.curiousmarc.com/instruments/hp-8082a-pulse-generator
* [8015A - A Pulse Generator for Today's Digital Circuits](https://www.hpl.hp.com/hpjournal/pdfs/IssuePDFs/1973-10.pdf)
* [Evolution of the Pulse Generator Product Line during the 1960s & 1970s](https://www.hpmemoryproject.org/wb_pages/wall_b_page_10e.htm)
* [Fundamentals of Time Interval Measurements](http://leapsecond.com/hpan/an200-3.pdf)
* [Precise Cable Length and Matching Measurements](https://www.hpmemoryproject.org/an/pdf/an_191-6.pdf)

# Footnotes
