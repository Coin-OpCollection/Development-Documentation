# KiCAD Notes

## Generating the PDFs

- [PCB Layout - footprints](/Toaplan2/Raizing/RA9704/ra9704-pcb_bnw_footprints.pdf)

    1. With PCB Editor open, navigate to File > Plot
    2. Put a check on exactly one non-copper (not ending in `.Cu`) single layer in the "Include Layers" pane (this will be the filename, `.pdf`). For example, F.Adhesive.
    3. Put a check on all non-copper (layers not ending in `.Cu`) layers in the "Plot on All Layers" pane.
    4. Choose "Black and white" in the drop-down under "PDF Options" "Output Mode".
    5. Click "Plot"
    6. Check the output for correctness.
    7. Rename/overwrite to `ra9704-pcb_bnw_footprints.pdf`.

- [PCB Layout - color top and bottom copper layers](/Toaplan2/Raizing/RA9704/ra9704-pcb_color_front_and_back.pdf)

    1. With PCB Editor open, navigate to File > Plot
    2. Use the same settings as (2) and (3) "PCB Layout - footprints" section, but also put a check on the copper layers `F.Cu` and `B.Cu` in the "Plot on All Layers" pane.
    3. Choose "Color" in the drop-down under "PDF Options" "Output Mode".
    4. Click "Plot"
    5. Check the output for correctness.
    6. Rename/overwrite to `ra9704-pcb_color_front_and_back.pdf`.
    
- [PCB Layout - color top copper plane](/Toaplan2/Raizing/RA9704/ra9704-In1_Cu.pdf)

    1. With PCB Editor open, navigate to File > Plot
    2. Put a check on ONLY `In1.Cu` the "Include Layers" pane.
    3. Uncheck everything in the "Plot on All Layers" pane.
    4. Choose "Color" in the drop-down under "PDF Options" "Output Mode".
    5. Click "Plot"
    6. Check the output for correctness. The filename is the same as the layer you selected, this time; no need to rename the file.

- [PCB Layout - color +5V copper plane](/Toaplan2/Raizing/RA9704/ra9704-In2_5V_Cu.pdf)
   
  Same as above for the "color top copper plane" layer, but choose "In2.5V.Cu".

- [PCB Layout - color GND copper plane](/Toaplan2/Raizing/RA9704/ra9704-In3_GND_Cu.pdf)

  Same as above, but choose "In3.GND.Cu".

- [PCB Layout - color +12V copper plane](/Toaplan2/Raizing/RA9704/ra9704-In4_12V_Cu.pdf)

  Same as above, but choose "In4.12V.Cu".
