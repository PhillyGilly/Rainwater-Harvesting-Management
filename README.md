# Rainwater Harvesting Management in Home Assistant

I have a rainwater collection system to provide "grey water" for flushing the toilets in my house.
The system supplied [Rainwater Harvesting UK](https://www.rainwaterharvesting.co.uk/rainwater-harvesting-gravity-feed-systems/) comprises of a 7,500l storage tank under the drive, a submersible pump and control panel, and a 40l header tank in the loft. The system supplied is basically a "fit-and-forget" in that it just works and if it isn't working it switches over to mains water. It does have LED status indicators on the control panel, but at the time of installation there was no remote monitoring capability. As a consequence when the pump failed aftert three years, I was unaware for some time that I was using expensive drinking water to flush my toilets.

Initially I considered implementing a small system that would monitoring the voltages on the control panel LEDs and realay those by wifi and mqtt to my Home Assistant, I discounted that as unecessarily invasive.to  

QDY30A-tank-level-sensor Work in Progress
I just found this on AliExpress: 
￡28.59 | 4-20mA 0-10V 0-5V RS485 Submerged Liquid Tank Stainless Steel Hydraulic Deep Well Water Level Sensor Transmitter
https://a.aliexpress.com/_EjQJvbW


![2025-01-30 17 16 27](https://github.com/user-attachments/assets/f091f2ce-fabc-4518-b8aa-d546361701b8)

![image](https://github.com/user-attachments/assets/0c1adee1-340c-4464-98e8-3765311c9c77)

