<script lang="ts">
  import { page } from '$app/stores';
  import PinFilled from 'carbon-icons-svelte/lib/PinFilled.svelte';
  import PageNav from '$lib/components/PageNav/index.svelte';
  import PrimaryButton from '$lib/components/PrimaryButton/index.svelte';
  import PageBody from '$lib/components/PageBody/index.svelte';
  import { group } from '$lib/components/Course/store';
  import { profile } from '$lib/utils/store/user';
  import CourseContainer from '$lib/components/CourseContainer/index.svelte';
  import RoleBasedSecurity from '$lib/components/RoleBasedSecurity/index.svelte';
  import { isNewFeedModal } from '$lib/components/Course/components/NewsFeed/store';
  import NewsFeedCard from '$lib/components/Course/components/NewsFeed/NewsFeedCard.svelte';
  import NewFeedModal from '$lib/components/Course/components/NewsFeed/NewFeedModal.svelte';
  import {
    createComment,
    deleteNewsFeedComment,
    deleteNewsFeed,
    fetchNewsFeeds,
    handleEditFeed,
    toggleFeedIsPinned,
    fetchNewsFeedReaction
  } from '$lib/utils/services/newsfeed';
  import { newsFeed } from '$lib/components/Course/components/NewsFeed/store';
  import Box from '$lib/components/Box/index.svelte';
  import { supabase } from '$lib/utils/functions/supabase';
  import { snackbar } from '$lib/components/Snackbar/store';
  import type { Feed } from '$lib/utils/types/feed';
  import {
    NOTIFICATION_NAME,
    triggerSendEmail
  } from '$lib/utils/services/notification/notification';
  import NewsFeedLoader from '$lib/components/Course/components/NewsFeed/NewsFeedLoader.svelte';
  export let data;

  let createdComment;
  let edit = false;
  let isLoading = false;
  let editFeed: Feed;
  let author = {
    id: '',
    username: '',
    fullname: '',
    avatar_url: ''
  };

  let query = new URLSearchParams($page.url.search);
  let feedId = query.get('feedId') || '';

  const onEdit = (id: string, content: string) => {
    handleEditFeed(id, content);

    $newsFeed = $newsFeed.map((feed) => {
      if (feed.id === id) {
        return { ...feed, content: content };
      }

      return feed;
    });
  };

  const deleteComment = (id: string) => {
    deleteNewsFeedComment(id);
    $newsFeed = $newsFeed.flatMap((feed) => ({
      ...feed,
      comment: feed.comment.filter((comment) => comment.id !== id)
    }));

    return snackbar.success('Comment Deleted');
  };

  const addNewReaction = async (reactionType: string, feedId: string, authorId: string) => {
    const { data } = await fetchNewsFeedReaction(feedId);

    if (!data) return;

    const reactedFeed = data || $newsFeed.find((feed) => feed.id === feedId);

    let reactedAuthorIds: string[] = reactedFeed.reaction[reactionType];

    if (reactedAuthorIds.includes(authorId)) {
      reactedAuthorIds = reactedAuthorIds.filter(
        (reactionAuthorId) => reactionAuthorId !== authorId
      );
    } else {
      reactedAuthorIds = [...reactedAuthorIds, authorId];
    }

    reactedFeed.reaction = {
      ...reactedFeed.reaction,
      [reactionType]: reactedAuthorIds
    };

    const response = await supabase
      .from('course_newsfeed')
      .update({
        reaction: reactedFeed.reaction
      })
      .eq('id', feedId);

    if (response.error) {
      return snackbar.error('An error occurred while reacting to news feed');
    }

    $newsFeed = $newsFeed.map((feed) => {
      if (feed.id === feedId) {
        feed.reaction = reactedFeed.reaction;
      }

      return feed;
    });
  };

  const addNewComment = async (comment: string, feedId: string, authorId: string) => {
    const { response } = await createComment({
      content: comment,
      author_id: authorId,
      course_newsfeed_id: feedId
    });

    if (response.error) {
      return snackbar.error('An error occurred while creating comment');
    }

    if (!response.data) return;

    createdComment = response?.data[0];

    triggerSendEmail(NOTIFICATION_NAME.NEWSFEED, {
      authorId: author.id,
      feedId: feedId,
      comment
    });

    $newsFeed = $newsFeed.map((feed) => {
      if (feed.id === feedId) {
        const newComment = {
          id: createdComment.id,
          author: {
            profile: { ...author }
          },
          created_at: createdComment.created_at,
          content: comment
        };

        snackbar.success('Comment added');

        return {
          ...feed,
          comment: [...feed.comment, { ...newComment }]
        };
      }

      return feed;
    });
  };

  const onPin = async (feedId, isPinned) => {
    const newIsPinned = !isPinned;
    const { response } = await toggleFeedIsPinned(feedId, newIsPinned);

    if (response.error) {
      return snackbar.error('Failed to toggle pinned feed');
    }

    $newsFeed = $newsFeed.map((feed) => {
      if (feed.id === feedId) {
        snackbar.success(`${newIsPinned ? 'Pinned' : 'Unpinned'} Successfully`);

        feed.isPinned = newIsPinned;
        return feed;
      }

      return feed;
    });
  };

  const deleteFeed = (id: string) => {
    deleteNewsFeed(id);

    const deletedFeed = $newsFeed.filter((feed) => feed.id !== id);
    snackbar.success('Feed deleted successfully');

    $newsFeed = deletedFeed;
  };

  const initNewsFeed = async (courseId: string) => {
    if (!courseId) return;
    try {
      isLoading = true;
      const { data, error } = await fetchNewsFeeds(courseId);
      if (error) {
        snackbar.error('Failed to fetch news feeds');
      }
      if (data) {
        $newsFeed = data.map((feedItem) => ({
          ...feedItem,
          isPinned: feedItem.is_pinned
        }));
      }
    } catch (error) {
      snackbar.error('An error occured while fetching feed');
    } finally {
      isLoading = false;
    }
  };

  function setAuthor(groups, profileId) {
    const currentGroupMember = groups.people.find((person) => person.profile_id === profileId);

    author = {
      id: currentGroupMember?.id || '',
      username: $profile.username || '',
      fullname: $profile.fullname || '',
      avatar_url: $profile.avatar_url || ''
    };
  }

  $: initNewsFeed(data.courseId);

  $: setAuthor($group, $profile.id);

  $: $newsFeed = $newsFeed.sort((a, b) => Number(b.isPinned) - Number(a.isPinned));
</script>

<CourseContainer bind:courseId={data.courseId}>
  <PageNav title="News Feed" disableSticky={true}>
    <slot:fragment slot="widget">
      <RoleBasedSecurity allowedRoles={[1, 2]}>
        <PrimaryButton className="mr-2" label="Add" onClick={() => ($isNewFeedModal.open = true)} />
      </RoleBasedSecurity>
    </slot:fragment>
  </PageNav>

  <PageBody width="max-w-4xl px-3">
    <RoleBasedSecurity allowedRoles={[1, 2]}>
      <NewFeedModal
        courseId={data.courseId}
        {author}
        bind:edit
        bind:editFeed
        onSave={(newFeed) => {
          $newsFeed = [newFeed, ...$newsFeed];
        }}
        {onEdit}
      />
    </RoleBasedSecurity>
    {#if isLoading}
      <div>
        <NewsFeedLoader />
        <NewsFeedLoader />
        <NewsFeedLoader />
      </div>
    {:else if !$newsFeed.length}
      <Box>
        <div class="flex justify-between flex-col items-center w-[90%] md:w-96">
          <img src="/images/empty-lesson-icon.svg" alt="Lesson" class="my-2.5 mx-auto" />
          <h2 class="text-xl my-1.5 font-normal">No post yet</h2>
          <p class="text-sm text-center text-slate-500">
            Make a post to your class. Start by clicking on the Add button.
          </p>
        </div>
      </Box>
    {:else}
      {#each $newsFeed as feed}
        {#if feed.isPinned}
          <div class="flex items-center gap-2 mb-3">
            <PinFilled size={16} />

            <p class="text-sm">Pinned</p>
          </div>
        {/if}
        <NewsFeedCard
          {feed}
          {deleteFeed}
          {addNewComment}
          {deleteComment}
          {addNewReaction}
          {onPin}
          {author}
          bind:edit
          bind:editFeed
          isActive={feedId === feed.id}
        />
      {/each}
    {/if}
  </PageBody>
</CourseContainer>
