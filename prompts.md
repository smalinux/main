
[pinctrl subsystem](pinctrl%20subsystem.md)
I will ask questions about pinctrl subsystem, in linux and barebox, when you talk to me, you are talking with good experienced engineer, not someone new, pinctrl in linux here
@/src/linux/drivers/pinctrl/ and barebox pinctrl here @/src/barebox/drivers/pinctrl/ just read what is exist in this repos and answer my questions in this file @pinctrl subsystem.md What is
the main things that barebox did from design perspective made pinctrl smaller than Linux?
I know that they don't parse DT on probe, they moved the DT parse when actual use happen
pluse in Linux first DT get parsed and they fill data structures and then there are other functions read these structures and use or send the actual use for writing to the hardware
what else they do in barebox? explain in details what happens
