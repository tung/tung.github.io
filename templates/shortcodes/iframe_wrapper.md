{# Usage: iframe_wrapper(relwidth=int, relheight=int, fullwidth=bool) #}
{% if fullwidth %}{% set pcw = 90 %}{% else %}{% set pcw = 55 %}{% endif %}
{% set pch1000 = pcw / relwidth * relheight * 1000 | round %}
{% set pch = pch1000 / 1000 %}
{% set pchsmall1000 = 100 / relwidth * relheight * 1000 | round %}
{% set pchsmall = pchsmall1000 / 1000 %}
<style type="text/css">
#iframe-wrapper-{{ nth }} {
  padding-bottom: {{ pch }}%;
}
@media (max-width: 760px) {
  #iframe-wrapper-{{ nth }} {
    padding-bottom: {{ pchsmall }}%;
  }
}
</style>
<figure id="iframe-wrapper-{{ nth }}" class="iframe-wrapper {% if fullwidth %}fullwidth{% endif %}">

{{- body -}}

</figure>
