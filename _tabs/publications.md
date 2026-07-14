---
icon: fas fa-book
order: 3
title: Publications
---

## Publications

{% assign featured_pubs = site.data.publications.publications | where: "featured", true | sort: "order" %}
{% assign all_pubs = site.data.publications.publications | sort: "order" | reverse %}

{% if all_pubs.size > 0 %}

## Featured Publications

{% for pub in featured_pubs %}

### {{ pub.title }}

**Authors**: {% for author in pub.authors %}{% if forloop.first %}{{ author }}{% else %}, {{ author }}{% endif %}{% endfor %}

**Journal**: {{ pub.journal }} ({{ pub.year }})  
**Status**: {{ pub.status }}

**Abstract**:
> {{ pub.abstract }}

**Keywords**: {% for keyword in pub.keywords %}{{ keyword }}{% if forloop.last %}.{% else %}, {% endif %}{% endfor %}

**Methods**:
{% for method in pub.methods %}
- {{ method }}
{% endfor %}

**Research Themes**:
{% for theme_id in pub.research_themes %}
  {% for theme in site.data.research_themes.themes %}
    {% if theme.id == theme_id %}
- {{ theme.name }}
    {% endif %}
  {% endfor %}
{% endfor %}

**Related Projects**:
{% for project_id in pub.related_projects %}
  {% for project in site.data.projects_data.projects %}
    {% if project.id == project_id %}
- {{ project.title }}
    {% endif %}
  {% endfor %}
{% endfor %}

**Citation**:
```
{{ pub.bibtex }}
```

**Links**:
{% if pub.doi %}
- [DOI: {{ pub.doi }}](https://doi.org/{{ pub.doi }})
{% endif %}
{% if pub.url %}
- [Journal Link]({{ pub.url }})
{% endif %}
{% if pub.pdf %}
- [PDF]({{ pub.pdf }})
{% endif %}
{% if pub.preprint %}
- [Preprint]({{ pub.preprint }})
{% endif %}

---

{% endfor %}

## All Publications

{% for pub in all_pubs %}
- **{{ pub.title }}** ({{ pub.year }}) — {{ pub.journal }} — [{{ pub.status }}]

{% endfor %}

{% else %}

## Research Manuscripts in Preparation

I am currently working on research manuscripts that will be submitted to peer-reviewed journals. Check back for updates on publication status.

**Current Work**:
- Palladium nanocluster stability and solvation effects (Journal of Chemical Physics)
- Additional research findings in progress

{% endif %}

---

## Preprints & Conference Presentations

Information about preprints and conference presentations will be added as they become available.

---

## Citation & Attribution

If you use my research in your work, please cite appropriately using the BibTeX entries above.

For questions about my research or citations, please [contact me](/contact/).

---

## Open Science

I am committed to open science practices:
- ✓ Code published on GitHub
- ✓ Data availability upon request
- ✓ Preprints available
- ✓ Reproducible research

I believe scientific knowledge should be accessible and reproducible.
