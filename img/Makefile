all: temporal_update_before.png temporal_update_goal.png temporal_update_after.png temporal_update_leftovers.png snapshot_query.png

snapshot_query.png: snapshot_query_layer1.png snapshot_query_layer2.png
	# `-transparent white` doesn't work, because we wind up with grayscale,
	# but `-compose multiply` is perfect, because `white times anything = anything`.
	# So it makes white effectively transparent:
	magick snapshot_query_layer1.png snapshot_query_layer2.png -compose multiply -composite $@

%.png: %.txt
	ditaa -o -E $<

.PHONY: all
