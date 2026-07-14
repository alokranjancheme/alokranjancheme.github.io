---
icon: fas fa-flask
order: 1
title: Research
hide: false
---

## Research Overview

My research focuses on understanding molecular behavior in complex chemical systems through computational methods, with a long-term vision of developing sustainable chemical processes that address global energy and environmental challenges.

**Research Philosophy**: Combine rigorous molecular-level understanding with chemical engineering principles. Use computational methods not as an end in themselves, but as a bridge between quantum mechanics and macroscopic phenomena.

**Sustainability Mission**: Chemistry and chemical engineering are essential for addressing climate change, energy transition, and circular economy. I want to develop computational tools that accelerate the design of green processes and catalysts.

---

## Research Areas

{% for theme in site.data.research_themes.themes %}

### {{ theme.name }}
{: #{{ theme.id }}}

**Research Focus**: {{ theme.short_name }}

{{ theme.long_description }}

**Key Tools & Software**:
{% for tool in theme.tools %}
- {{ tool }}
{% endfor %}

**Technical Skills**:
{% for skill in theme.skills %}
- {{ skill }}
{% endfor %}

**Related Research Interests**:
{% for keyword in theme.keywords %}
- {{ keyword }}
{% endfor %}

**Future Research Directions**:
{{ theme.future_direction }}

---

{% endfor %}

## Current Research Focus (2024-2026)

**Primary Project**: Comprehensive molecular dynamics study of palladium nanoclusters (Pd₃ to Pd₅₅) in multiple solvent environments

**Key Findings**:
- Solvation effects dramatically influence cluster stability
- Different aggregation pathways in various solvents
- Water's hydrogen bonding network plays a stabilizing role
- Aggregation dynamics can be tracked using graph algorithms

**Methodology**:
- 90+ independent MD trajectories (100 ns each)
- Custom force field development and validation
- Advanced trajectory analysis using Python + MDAnalysis
- Machine learning for property prediction

**Publication Status**: Manuscript in preparation for Journal of Chemical Physics

---

## Research Interests

Beyond my current projects, I am interested in:

- **Advanced Sampling Techniques**: Enhanced MD, rare event sampling, metadynamics
- **Reactive Molecular Dynamics**: For studying catalytic reactions at the atomic level
- **Machine Learning for Molecular Systems**: Graph neural networks, transferable force fields, property prediction
- **Multiscale Modeling**: Connecting quantum mechanics → molecular scale → continuum scale
- **Green Hydrogen Technologies**: Catalyst design, process optimization, efficiency improvement
- **Biomass Valorization**: Converting biomass to chemicals and fuels
- **Process Intensification**: Novel reactor designs, efficient synthesis routes
- **High-Performance Computing**: GPU acceleration, scalability, exascale simulations
- **Open Source Scientific Software**: Tools for reproducible research

---

## Collaboration & Mentorship

I am interested in collaborating on projects involving:
- Computational chemistry and molecular dynamics
- Machine learning applications to chemistry
- Green chemistry and sustainable processes
- Scientific software development

For collaboration inquiries, please [contact me](/contact/).

---

## Publications

See my [publications page](/publications/) for papers and manuscripts.
