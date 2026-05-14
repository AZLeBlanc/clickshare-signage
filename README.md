Currently a work-in-progress proof of concept

PROPOSED STRUCTURE

clickshare-signage/
├── .github/
│   └── workflows/
│       └── update-manifests.yml    # Modified to handle all units
├── units/
│   ├── conference-room-a/
│   │   ├── images/
│   │   │   ├── 01-welcome.jpg
│   │   │   └── 02-safety.jpg
│   │   ├── index.html              # Slideshow for this unit
│   │   └── images.json             # Auto-generated manifest
│   ├── lobby/
│   │   ├── images/
│   │   │   ├── 01-brand.jpg
│   │   │   └── 02-awards.jpg
│   │   ├── index.html
│   │   └── images.json
│   ├── boardroom/
│   │   ├── images/
│   │   │   └── executive-message.jpg
│   │   ├── index.html
│   │   └── images.json
│   └── training-room/
│       ├── images/
│       ├── index.html
│       └── images.json
├── .nojekyll
└── README.md
