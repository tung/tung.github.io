<label for="note-{{ nth }}" class="margin-toggle{% if num %} sidenote-number{% endif %}"></label><input type="checkbox" id="note-{{ nth }}" class="margin-toggle"/><span class="{% if num %}sidenote{% else %}marginnote{% endif %}">
{{- body | markdown(inline=true) | safe -}}
</span>