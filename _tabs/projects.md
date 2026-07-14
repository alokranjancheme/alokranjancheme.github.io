---
icon: fas fa-project-diagram
order: 2
title: Projects
---

## Projects

All my projects are organized by research area. Each project describes the scientific problem, approach, methods, results, and impact.

---

{% for theme in site.data.research_themes.themes %}

## {{ theme.name }}

{{ theme.description }}

{% assign theme_projects = site.data.projects_data.projects | where: "theme", theme.id | sort: "order" %}

{% for project in theme_projects %}

### {{ project.title }}

**{{ project.subtitle }}**

**Problem**:  
{{ project.problem }}

**Approach**:  
{{ project.approach }}

**Methods**:
{% for method in project.methods %}
- {{ method }}
{% endfor %}

**Results**:  
{{ project.results }}

**Tools & Technologies**:
{% for tool in project.tools %}
- {{ tool }}
{% endfor %}

**Skills Developed**:
{% for skill in project.skills %}
- {{ skill }}
{% endfor %}

**Impact**:  
{{ project.impact }}

{% if project.links %}
  {% if project.links.blog %}
  [Read the blog post]({{ project.links.blog }})
  {% endif %}
{% endif %}

---

{% endfor %}

{% endfor %}

## Project Categories by Research Impact

### Machine Learning & Automation
- ML for molecular property prediction
- Automated data pipeline development
- Scientific computing frameworks

### Molecular Engineering & Simulation
- MD simulation methodology
- Force field development
- Trajectory analysis at scale

### Process Engineering & Sustainability
- Industrial hydrogen production
- Process modeling and optimization
- Computational thermodynamics

### High-Performance Computing
- GPU acceleration
- HPC workflow optimization
- Scalable scientific software

---

## Methodological Strengths

**Molecular Dynamics Simulation**:
- System building and parameterization
- Equilibration protocols (NVT, NPT)
- Production simulations with advanced sampling
- Trajectory analysis and data extraction

**Data Analysis & Scientific Computing**:
- Python-based scientific pipelines
- Large-scale data processing
- Statistical analysis
- Visualization and publication-quality graphics

**Machine Learning**:
- Deep learning for molecular data
- Feature engineering for chemistry
- Model validation and uncertainty quantification
- Predictive modeling

**HPC & Performance Optimization**:
- Batch job scheduling (SLURM, PBS)
- GPU computing and CUDA
- Code profiling and optimization
- Linux system administration

---

## Code & Resources

Most of my research code is available on GitHub:
[github.com/alokranjancheme](https://github.com/alokranjancheme)

All code is written with reproducibility in mind:
- Version controlled (Git)
- Documented
- Tested
- Published alongside findings

---

## Interested in Collaboration?

If you're interested in any of these projects or related research, please [get in touch](/contact/).
