# Missing data and unbuilt parts

Every `[bracket]` in the recipe, grouped by how it gets filled. Tick items off as the recipe is
completed.

## Measure on the reference unit (needs a wall meter or root)

- [ ] Wall power: idle, chat, image render, Q4_0 video, coding mode, HQ video mode; yearly energy estimate (`01-hardware`, `11-benchmarks`)
- [ ] GPU board power per mode, and CPU package power (RAPL needs root) (`01-hardware`)
- [ ] RAM speed, DIMM count, channels (`sudo dmidecode -t memory`) (`01-hardware`)
- [ ] Electrical width of each GPU slot (`sudo lspci -vv`) (`01-hardware`)
- [ ] PSU wattage from the label (`01-hardware`)
- [ ] Thermals in HQ video mode; room temperature; noise at idle and under load (`01-hardware`)
- [ ] Wi-Fi adapter model and throughput (`01-hardware`)
- [ ] Cold-boot time to enter HQ video mode (`06-modes`)
- [ ] Gemma 4 E2B speed with the CPU to itself (`04-llm-serving`, `11-benchmarks`)

## Benchmarks still to run

- [ ] LTX Q8 at 3, 4 and 5 seconds (`05-images-and-video`, `11-benchmarks`)
- [ ] Q4_0 vs Q8_0 video quality, same prompt and seed (`05`, `11`)
- [ ] Coding contexts filled to 128K (27B) and 160K (35B) (`04`, `11`)
- [ ] 3 to 8 concurrent chat users; 3+ concurrent research users (`07`, `11`)
- [ ] Whether public search engines rate-limit SearXNG under heavy use (`07`)

## Tests of things that are built

- [ ] Use from outside the home: phone on mobile data, laptop on a hotspot (`08`)
- [ ] Double-click each desktop icon over Moonlight (`08`)
- [ ] The watchdog handling a real GPU fault in service (`09`)
- [ ] The whole first-day sequence on a unit with no monitor; dummy display plug (`02`, `10`)
- [ ] iPhone: notifications (ntfy upstream decision) and the UI (`08`)

## Decisions for the owner

- [ ] Licence for this documentation (`README`)
- [ ] Publish a sanitised copy of the service code, or keep the recipe as description only (`03`)
- [ ] Model licences: redistribution on a gifted drive, commercial use of renders (`05`, `10`)
- [ ] Screen-lock policy with desktop auto-login (`02`)
- [ ] Cable addressing for the direct-cable setup (`10`)
- [ ] Success criteria for the gift (`10`)

## Not built yet

- [ ] Login for the status and files page (must happen before a unit leaves the house) (`03`, `08`)
- [ ] BIOS "power on after AC loss" (`09`)
- [ ] Alerts when chat or search stop answering; backups; disk-space check; burn-in procedure (`09`)
- [ ] macOS and Windows laptop setup scripts (`04`, `08`)
- [ ] Image-to-video from an approved render; 30-second film recipe; optional x2 upscaler (`05`)
- [ ] Multi-step research agent (`07`)
- [ ] Install script and fleet clone process (`10`, `PROMPT-STAGES` stage 11)
- [ ] Remaining user guides, owner's manual, learning path (`10`, stage 10)
- [ ] Offer the stable-diffusion.cpp patch upstream (`05`)
- [ ] A clean way to stop a runaway Open WebUI generation (`12`)

## Per fleet unit

- [ ] Chassis, chipset, BIOS version, CPU generation, slot layout (`01`)
- [ ] Vulkan device order and PCI addresses (`02`)
- [ ] Laptop model; USB-Ethernet adapter model (`01`, `10`)
- [ ] Used GPU price at build time (`01`)
